# app/index.html — part 3 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 4955–6864 of app/index.html.

    if (!d.findings.length) h += "<p style='color:var(--dim)'>No findings yet.</p>";
    d.findings.forEach(function (f) {
      h += "<div class='card' style='margin-top:8px'><b>" + esc(f.title) + "</b> <span class='mono' style='font-size:11px;color:" + caseSevColor(f.severity) + "'>" + esc(f.severity.toUpperCase()) + "</span>" + (f.internal ? " <span class='mono' style='font-size:11px;color:#f5b544'>INTERNAL</span>" : "") + (f.author === "navi" ? " <span class='mono' style='font-size:11px;color:#34d399'>FILED BY NAVI</span>" : "") + (f.state === "lead" ? " <span class='mono' style='font-size:11px;color:var(--am)'>LEAD — UNVERIFIED</span>" : " <span class='mono' style='font-size:11px;color:var(--dim)'>CONFIRMED</span>") + "<br><span style='color:var(--dim);font-size:12px'>" + esc(f.filed) + " · " + esc(f.file) + "</span><div class='md' style='margin-top:4px'>" + mdToHtml(f.body, "cases/" + d.slug + "/") + "</div>"
        + (f.state === "lead" ? "<div class='row' style='margin-top:6px'><button class='act' data-cf-review='findings' data-cf-key='" + esc(f.file) + "' data-cf-action='promote' style='min-height:44px'>Promote to confirmed</button><button class='ghost' data-cf-review='findings' data-cf-key='" + esc(f.file) + "' data-cf-action='strike' style='min-height:44px;color:#f87171'>Strike it</button></div>" : "")
        // The delete door (v0.5.9e, #280): every finding
        // carries one. A finding born from a dossier element
        // deletes the ELEMENT — from the dossier, every case
        // it stands in, the Vault formations — and the arm
        // strip says so before it fires; any other finding
        // is removed from this case alone. The case's
        // custody record keeps the line either way.
        + (cfDelArm === f.file
          ? "<div class='card' style='margin-top:6px;border-color:#f87171'><p style='font-size:13px'>"
            + (f.element
              ? "Delete <b>" + esc(f.title) + "</b>? It is a dossier element — it also leaves " + (f.homes && f.homes.dossier ? "the dossier <b>" + esc(f.homes.dossier) + "</b>" : "its dossier") + (f.homes && f.homes.cases && f.homes.cases.length > 1 ? ", every case it stands in (" + f.homes.cases.length + ")" : "") + " and the Vault formations. Every case it leaves keeps a custody line naming this case as where it was deleted from."
              : (f.elements && f.elements.length
                ? "Delete <b>" + esc(f.title) + "</b>? This filing carries " + f.elements.length + " dossier element(s) into this case. Deleting the filing removes it from this case and its formed Vault document only — the elements stay in their dossier, where each can be deleted on its own."
                : "Delete <b>" + esc(f.title) + "</b>? It leaves this case and its formed Vault document; nothing else is touched. The case's custody record keeps the line."))
            + "</p><div class='row' style='margin-top:6px'><input class='grow' id='cf-del-reason' type='text' placeholder='Reason (optional) — irrelevant / repeat / incorrect…' aria-label='Reason for deleting (optional)'><button class='ghost' data-cf-delfinding-ok='" + esc(f.file) + "' style='min-height:44px;color:#f87171'>Delete</button><button class='ghost' data-cf-delfinding-no='1' style='min-height:44px'>Keep it</button></div></div>"
          : "<div class='row' style='margin-top:6px'><button class='ghost' data-cf-delfinding='" + esc(f.file) + "' style='min-height:44px' aria-label='Delete finding " + esc(f.title) + "'>Delete</button></div>")
        + "</div>";
    });
    h += "</div>";
    // Entities (Build 2, #199): the cast register. Navi's
    // auto-logged entries wear their LEAD badge and their
    // provenance; review is one tap.
    h += "<div class='card'><h3>Entities</h3>";
    h += "<div class='row'><input class='grow' id='cf-ent-name' type='text' placeholder='Name — a person, org, account, system, or place' aria-label='Entity name'><select id='cf-ent-type' aria-label='Entity type'><option value='person'>person</option><option value='org'>org</option><option value='account'>account</option><option value='system'>system</option><option value='place'>place</option><option value='other'>other</option></select><input class='grow' id='cf-ent-role' type='text' placeholder='Role in the case' aria-label='Entity role'></div>";
    h += "<div class='row' style='margin-top:6px'><button class='act' id='cf-ent-add' style='min-height:44px'>Add the entity</button></div>";
    if (!(d.entities || []).length) h += "<p style='color:var(--dim)'>No entities yet — the cast assembles here.</p>";
    (d.entities || []).forEach(function (e) {
      var f = e.fields || {};
      h += "<div class='card' style='margin-top:8px'><b>" + esc(e.heading) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(f.type || "other") + "</span>" + (f.author === "navi" ? " <span class='mono' style='font-size:11px;color:#34d399'>LOGGED BY NAVI</span>" : "") + (f.state === "lead" ? " <span class='mono' style='font-size:11px;color:var(--am)'>LEAD — UNVERIFIED</span>" : " <span class='mono' style='font-size:11px;color:var(--dim)'>CONFIRMED</span>")
        + (f.role ? "<br><span style='font-size:13px'>" + esc(f.role) + "</span>" : "")
        + (f.aliases ? "<br><span style='color:var(--dim);font-size:12px'>aliases: " + esc(f.aliases) + "</span>" : "")
        + (f.identifiers ? "<br><span style='color:var(--dim);font-size:12px'>identifiers: " + esc(f.identifiers) + "</span>" : "")
        + "<br><span style='color:var(--dim);font-size:12px'>" + esc(f.filed || "") + (f.source ? " · " + esc(f.source) : "") + "</span>"
        + (e.body ? "<div class='md' style='margin-top:4px'>" + mdToHtml(e.body, "cases/" + d.slug + "/") + "</div>" : "")
        + (f.state === "lead" ? "<div class='row' style='margin-top:6px'><button class='act' data-cf-review='entities' data-cf-key='" + esc(e.key) + "' data-cf-action='promote' style='min-height:44px'>Promote to confirmed</button><button class='ghost' data-cf-review='entities' data-cf-key='" + esc(e.key) + "' data-cf-action='strike' style='min-height:44px;color:#f87171'>Strike it</button></div>" : "")
        + "</div>";
    });
    h += "</div>";
    // Observables (Build 2, #199): the IOC register — also
    // the Wire/TIC case-terms source.
    h += "<div class='card'><h3>Observables</h3>";
    h += "<div class='row'><input class='grow' id='cf-obs-value' type='text' placeholder='Value — an IP, domain, URL, hash, file, wallet…' aria-label='Observable value'><select id='cf-obs-kind' aria-label='Observable kind'><option value='ip'>ip</option><option value='domain'>domain</option><option value='url'>url</option><option value='hash'>hash</option><option value='file'>file</option><option value='wallet'>wallet</option><option value='other'>other</option></select><input class='grow' id='cf-obs-ctx' type='text' placeholder='Context — where it was seen' aria-label='Observable context'></div>";
    h += "<div class='row' style='margin-top:6px'><button class='act' id='cf-obs-add' style='min-height:44px'>Add the observable</button></div>";
    if (!(d.observables || []).length) h += "<p style='color:var(--dim)'>No observables yet. What lands here feeds the Wire's case matching.</p>";
    (d.observables || []).forEach(function (e) {
      var f = e.fields || {};
      h += "<div class='card' style='margin-top:8px'><span class='mono'><b>" + esc(e.heading) + "</b></span> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(f.kind || "other") + "</span>" + (f.author === "navi" ? " <span class='mono' style='font-size:11px;color:#34d399'>LOGGED BY NAVI</span>" : "") + (f.state === "lead" ? " <span class='mono' style='font-size:11px;color:var(--am)'>LEAD — UNVERIFIED</span>" : " <span class='mono' style='font-size:11px;color:var(--dim)'>CONFIRMED</span>")
        + "<br><span style='color:var(--dim);font-size:12px'>" + esc(f.filed || "") + (f.source ? " · " + esc(f.source) : "") + "</span>"
        + (e.body ? "<div class='md' style='margin-top:4px'>" + mdToHtml(e.body, "cases/" + d.slug + "/") + "</div>" : "")
        + (f.state === "lead" ? "<div class='row' style='margin-top:6px'><button class='act' data-cf-review='observables' data-cf-key='" + esc(e.key) + "' data-cf-action='promote' style='min-height:44px'>Promote to confirmed</button><button class='ghost' data-cf-review='observables' data-cf-key='" + esc(e.key) + "' data-cf-action='strike' style='min-height:44px;color:#f87171'>Strike it</button></div>" : "")
        + "</div>";
    });
    h += "</div>";
    // Tasks (Build 2, #199): next steps — Navi proposes, the
    // operator checks off.
    h += "<div class='card'><h3>Tasks</h3>";
    h += "<div class='row'><input class='grow' id='cf-task-text' type='text' placeholder='A next step — e.g. Pull the mailbox audit log' aria-label='Task text'></div>";
    h += "<div class='row' style='margin-top:6px'><button class='act' id='cf-task-add' style='min-height:44px'>Add the task</button></div>";
    if (!(d.tasks || []).length) h += "<p style='color:var(--dim)'>No tasks yet.</p>";
    (d.tasks || []).forEach(function (e) {
      var f = e.fields || {};
      var done = String(f.done || "").toLowerCase() === "yes";
      // v0.6.6e (#330): a task standing as her PROPOSAL is
      // settled, not checked off — Approve opens it as a
      // standing task, Dismiss strikes it (the strike stands
      // on the case's timeline). The badge names the state
      // the register carries, never a guess from authorship.
      var proposed = String(f.state || "").toLowerCase() === "proposed";
      h += "<div class='row' style='margin-top:6px'><span class='grow' style='font-size:13px;" + (done ? "color:var(--dim);text-decoration:line-through" : "") + "'>" + esc(e.heading) + (proposed ? " <span class='mono' style='font-size:11px;color:#34d399'>PROPOSED BY NAVI — AWAITING APPROVAL</span>" : (f.author === "navi" ? " <span class='mono' style='font-size:11px;color:#34d399'>PROPOSED BY NAVI</span>" : "")) + "</span>"
        + (proposed
          ? "<button class='act' data-cf-review='tasks' data-cf-key='" + esc(e.key) + "' data-cf-action='approve' style='min-height:44px'>Approve</button><button class='ghost' data-cf-review='tasks' data-cf-key='" + esc(e.key) + "' data-cf-action='dismiss' style='min-height:44px;color:#f87171'>Dismiss</button>"
          : "<button class='ghost' data-cf-review='tasks' data-cf-key='" + esc(e.key) + "' data-cf-action='" + (done ? "reopen" : "done") + "' style='min-height:44px'>" + (done ? "Reopen" : "Done") + "</button>")
        + "</div>";
    });
    h += "</div>";
    // Evidence
    h += "<div class='card'><h3>Evidence</h3>";
    h += "<div id='cf-drop' tabindex='0' role='button' aria-label='Drop evidence files here or activate to upload' style='border:1px dashed " + (caseDragOver ? "var(--accent)" : "var(--line)") + ";border-radius:8px;padding:16px 12px;text-align:center;cursor:pointer'><b>" + (caseDragOver ? "Drop the evidence to file it" : "Drag files here") + "</b><br><span style='color:var(--dim);font-size:12px'>anything the case needs to keep — or click to upload</span><input id='cf-files' type='file' multiple style='display:none' aria-label='Upload evidence files'></div>";
    if (!d.evidence.length) h += "<p style='color:var(--dim)'>No evidence files yet.</p>";
    d.evidence.forEach(function (ev) {
      h += "<div style='margin-top:8px'><div class='row'><span class='mono grow' style='font-size:12px'>" + esc(ev.name) + "</span><span style='color:var(--dim);font-size:12px'>" + ev.size + " bytes</span></div>"
        + (ev.sha256 ? "<div class='mono' style='color:var(--dim);font-size:11px;word-break:break-all'>SHA-256 " + esc(ev.sha256) + "</div>" : "<div style='color:var(--am);font-size:12px'>Not yet in the custody register — no hash on file.</div>")
        + (ev.description ? "<div style='font-size:13px;margin-top:2px'>" + esc(ev.description) + "</div>" : "")
        + (ev.acquired ? "<div style='color:var(--dim);font-size:12px'>Acquired: " + esc(ev.acquired) + "</div>" : "")
        + (ev.custody ? "<div style='color:var(--dim);font-size:12px'>Custody: " + esc(ev.custody) + "</div>" : "")
        + "</div>";
    });
    h += "</div>";
    // Timeline
    h += "<div class='card'><h3>Timeline</h3>";
    h += "<div class='row'><input class='grow' id='cf-t-title' type='text' placeholder='Entry title — e.g. Kickoff call with the client' aria-label='Timeline entry title'></div>";
    h += "<textarea id='cf-t-body' rows='3' placeholder='What happened.' aria-label='Timeline entry detail' style='margin-top:6px'></textarea>";
    h += "<div class='row' style='margin-top:6px'><button class='act' id='cf-t-add' style='min-height:44px'>Add the entry</button></div>";
    if (!d.timeline.length) h += "<p style='color:var(--dim)'>No entries yet.</p>";
    d.timeline.forEach(function (e) { h += "<div style='margin-top:10px'><b>" + esc(e.title) + "</b> <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(e.date) + "</span><div class='md' style='color:var(--dim)'>" + mdToHtml(e.body, "cases/" + d.slug + "/") + "</div></div>"; });
    h += "</div>";
    // Delete this case (Build 15, #163): complete, and confirmed
    // in words before it happens — the confirm names the case,
    // everything of it in the Vault, and its tied chat, and
    // states the backup truth. No one-tap accidents.
    h += "<div class='card' style='border-color:#f87171'><h3 style='color:#f87171'>Delete this case</h3>";
    if (caseDeleteArm === d.slug) {
      h += "<p>Deleting <b>" + esc(d.name) + "</b> removes all of it, for good:</p>";
      h += "<p style='color:var(--dim)'>· the case itself and <b>every file it owns in the Vault</b> — " + d.findings.length + " finding" + (d.findings.length === 1 ? "" : "s") + ", " + d.evidence.length + " evidence file" + (d.evidence.length === 1 ? "" : "s") + ", " + d.timeline.length + " timeline entr" + (d.timeline.length === 1 ? "y" : "ies") + ", its manifest and any exported report;<br>· <b>its tied chat</b> — the whole case conversation goes with it.</p>";
      h += "<p style='color:var(--dim)'>Backups already taken keep the copies inside them — deleting here does not reach into backup history.</p>";
      h += "<div class='row' style='margin-top:8px'><button class='act' id='cf-del-yes' style='min-height:44px;background:#f87171;border-color:#f87171;color:#14100f'>Delete the case, its files, and its chat</button> <button class='ghost' id='cf-del-no' style='min-height:44px'>Keep the case</button></div>";
    } else {
      h += "<p style='color:var(--dim)'>Removing a case removes all of its files from the Vault as well — and its tied chat. There is no undo here; backups already taken keep their snapshot copies.</p>";
      h += "<div class='row' style='margin-top:8px'><button class='ghost' id='cf-del' style='min-height:44px;color:#f87171'>Delete this case…</button></div>";
    }
    h += "</div>";
    return h;
  }
  // The list (cold start lands here, always — #114's rule).
  if (!casesState && !casesFetching) refreshCases();
  h += "<div class='card'><h3>Open a case</h3><div class='row'><input class='grow' id='cf-name' type='text' placeholder='Case name — e.g. Q3 external assessment' aria-label='Case name'></div><div class='row' style='margin-top:6px'><input class='grow' id='cf-client' type='text' placeholder='Client' aria-label='Case client'><input class='grow' id='cf-scope' type='text' placeholder='Scope — e.g. external, one host' aria-label='Case scope'></div>"
    + "<div class='row' style='margin-top:6px'><input class='grow' id='cf-summary' type='text' placeholder='Summary — one line: what this case is and where it stands' aria-label='Case summary'></div>"
    + "<div class='row' style='margin-top:6px'><select id='cf-type' aria-label='Case type'><option value='incident'>Incident</option><option value='investigation' selected>Investigation</option><option value='research'>Research</option><option value='hunt'>Hunt</option><option value='locate'>Locate</option></select><select id='cf-priority' aria-label='Case priority'><option value='low'>Low priority</option><option value='normal' selected>Normal priority</option><option value='high'>High priority</option><option value='critical'>Critical priority</option></select><input class='grow' id='cf-tags' type='text' placeholder='Tags — comma separated' aria-label='Case tags'><label style='color:var(--dim);font-size:13px'><input type='checkbox' id='cf-sensitive'> Sensitive</label></div>"
    + "<div class='row' style='margin-top:6px'><button class='act' id='cf-create' style='min-height:44px'>Create the case</button></div></div>";
  if (casesFetching && !casesState) h += "<p style='color:var(--dim)'>Reading the cases from the desk…</p>";
  (casesState || []).forEach(function (c) {
    var delArm = caseListDeleteArm === c.slug;
    h += "<div class='card' style='margin-top:8px'><b>" + esc(c.name) + "</b>" + (caseOpenSlug === c.slug ? " <span class='mono' style='font-size:11px;color:#34d399'>OPEN</span>" : "") + (c.sensitive ? " <span class='mono' style='font-size:11px;color:#f87171'>SENSITIVE</span>" : "") + (c.priority && c.priority !== "normal" ? " <span class='mono' style='font-size:11px;color:" + caseSevColor(c.priority === "critical" ? "critical" : c.priority === "high" ? "high" : "low") + "'>" + esc(c.priority.toUpperCase()) + "</span>" : "") + (c.unverified ? " <span class='mono' style='font-size:11px;color:var(--am)'>" + c.unverified + " UNVERIFIED</span>" : "")
      + (c.summary ? "<br><span style='font-size:13px'>" + esc(c.summary) + "</span>" : "")
      + "<br><span style='color:var(--dim);font-size:12px'>" + esc(c.client || "—") + " · " + esc(c.status) + (c.type ? " · " + esc(c.type) : "") + " · opened " + esc(c.opened) + (c.status === "closed" && c.outcome ? " · outcome: " + esc(c.outcome) : "") + "</span>"
      + (c.tags && c.tags.length ? "<br><span style='color:var(--dim);font-size:12px'>tags: " + esc(c.tags.join(", ")) + "</span>" : "")
      + "<br><span style='color:var(--dim);font-size:12px'>" + c.findings + " finding" + (c.findings === 1 ? "" : "s") + " · " + c.evidence + " evidence · " + c.timeline + " timeline" + (typeof c.entities === "number" ? " · " + c.entities + " entities · " + c.observables + " observables · " + c.tasksOpen + " open tasks" : "") + "</span><div class='row' style='margin-top:6px'><button class='act' data-case-open='" + esc(c.slug) + "' style='min-height:44px'>Open the case</button>" +
      (delArm ? "" : "<button class='ghost' data-case-del='" + esc(c.slug) + "' style='min-height:44px;color:#f87171'>Delete</button>") + "</div>" +
      (delArm ? "<div style='margin-top:8px;border-top:1px solid var(--line);padding-top:8px'><span style='color:#f5b544;font-size:12px'>Delete <b>" + esc(c.name) + "</b>? That removes the case and every file it owns in the Vault — " + c.findings + " finding" + (c.findings === 1 ? "" : "s") + ", " + c.evidence + " evidence file" + (c.evidence === 1 ? "" : "s") + ", " + c.timeline + " timeline entr" + (c.timeline === 1 ? "y" : "ies") + " — and its tied chat. The deletion travels with Sync, so it won't come back from your other machine; copies inside backups already taken stay in those backups.</span><div class='row' style='margin-top:6px'><button class='act' data-case-del-yes='" + esc(c.slug) + "' data-case-name='" + esc(c.name) + "' style='min-height:44px;background:#f87171;border-color:#f87171;color:#14100f'>Delete the case</button><button class='ghost' data-case-del-no='1' style='min-height:44px'>Keep it</button></div></div>" : "") +
      "</div>";
  });
  if (casesState && !casesState.length) h += "<p style='color:var(--dim)'>No cases yet — the desk is clear. Open the first one above.</p>";
  return h;
}
function casePost(path, body, done) {
  fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) })
    .then(function (r) { return r.json(); })
    .then(function (j) { done(j || {}); })
    .catch(function () { caseMsg = "The bridge didn't answer — nothing was filed."; render(); });
}
function jumpToCaseChat(slug, onFail) {
  // The one road into a case's own chat (Build 15, #161): the
  // bridge ensures the bound thread and hands back its record;
  // the app seats it in the store and lands on it. The case
  // page's "Open the case chat" walks it, and so does the
  // intake receipt's button (v0.4.5e, #241) — one path, rigged
  // the same way in both places, never a decorative door.
  casePost("/api/cases/chat", { case: slug }, function (j) {
    if (j.ok && j.chat) {
      var rec = j.chat;
      if (!chats.some(function (c) { return c.id === rec.id; })) chats.push(rec);
      activeChatId = rec.id; messages = rec.messages; persist(); landOnChat();
    } else if (onFail) {
      onFail(j || {});
    }
  });
}
// --- New case from chat (v0.4.5e, #241, Route B) ----------------
// The deterministic companion to the conversational intake:
// the composer's New case button asks the bridge to swipe the
// active conversation into a draft (/api/cases/draft), shows
// it as a form the operator confirms or corrects, and on
// submit composes the ordinary doors — create, the lead door
// for the checked register entries, open, and the jump into
// the case's own chat. Every refusal lands on the form itself.
function ncPost(path, body, done, fail) {
  fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) })
    .then(function (r) { return r.json(); })
    .then(function (j) { done(j || {}); })
    .catch(function () { if (fail) fail(); });
}
function openNewCaseForm() {
  newCaseState = { busy: true, submitting: false, note: "", name: "", client: "", scope: "", summary: "", type: "", priority: "", tags: "", sensitive: false, entities: [], observables: [] };
  render();
  ncPost("/api/cases/draft", { chatId: activeChatId }, function (j) {
    if (!newCaseState) return; // closed while she swiped
    newCaseState.busy = false;
    if (j.ok && j.draft) {
      var dr = j.draft;
      newCaseState.name = dr.name || ""; newCaseState.client = dr.client || "";
      newCaseState.scope = dr.scope || ""; newCaseState.summary = dr.summary || "";
      newCaseState.type = dr.type || ""; newCaseState.priority = dr.priority || "";
      newCaseState.tags = dr.tags || ""; newCaseState.sensitive = !!dr.sensitive;
      newCaseState.entities = (dr.entities || []).map(function (e) { return { name: e.name || "", type: e.type || "other", role: e.role || "", on: true }; });
      newCaseState.observables = (dr.observables || []).map(function (o) { return { value: o.value || "", kind: o.kind || "other", context: o.context || "", on: true }; });
    } else {
      newCaseState.note = j.error || "The draft could not be read — the form is blank, and the fields are yours to fill.";
    }
    render();
  }, function () {
    if (!newCaseState) return;
    newCaseState.busy = false;
    newCaseState.note = "The bridge didn't answer, so there is no swipe — the form is blank, and the fields are yours to fill.";
    render();
  });
}
function newCaseFormHtml() {
  var st = newCaseState;
  if (!st) return "";
  var typeOpts = ["incident", "investigation", "research", "hunt", "locate"].map(function (t) { return "<option value='" + t + "'" + (st.type === t ? " selected" : "") + ">" + t + "</option>"; }).join("");
  var prioOpts = ["low", "normal", "high", "critical"].map(function (t) { return "<option value='" + t + "'" + (st.priority === t ? " selected" : "") + ">" + t + "</option>"; }).join("");
  var h = "<div id='nc-shade'></div>";
  h += "<div id='nc-dialog' role='dialog' aria-modal='true' aria-label='Start a new case from this chat'>";
  h += "<div style='display:flex;gap:8px;align-items:flex-start;justify-content:space-between'><div><b style='font-size:15px'>New case — from this chat</b><div style='color:var(--dim);font-size:12px;margin-top:2px'>Navi swiped this conversation for the shape of the case. Confirm or correct it — nothing is created until you start it, and what she swiped lands as unverified leads for your review.</div></div><button class='ghost' data-nc-cancel aria-label='Close the new case form' style='min-height:44px;flex:0 0 auto'>Close</button></div>";
  if (st.busy) h += "<p style='margin-top:10px;color:var(--dim)'>Navi is swiping this conversation…</p>";
  if (st.note) h += "<p style='margin-top:10px;color:var(--am)'>" + esc(st.note) + "</p>";
  h += "<div class='row' style='margin-top:10px'><input class='grow' id='nc-name' type='text' placeholder='Case name — required' aria-label='Case name' value='" + esc(st.name) + "'></div>";
  h += "<div class='row' style='margin-top:8px'><input class='grow' id='nc-client' type='text' placeholder='Client' aria-label='Client' value='" + esc(st.client) + "'><input class='grow' id='nc-scope' type='text' placeholder='Scope' aria-label='Scope' value='" + esc(st.scope) + "'></div>";
  h += "<div class='row' style='margin-top:8px'><input class='grow' id='nc-summary' type='text' placeholder='Summary — one or two lines' aria-label='Summary' value='" + esc(st.summary) + "'></div>";
  h += "<div class='row' style='margin-top:8px'><select id='nc-type' aria-label='Case type'>" + typeOpts + "</select><select id='nc-priority' aria-label='Priority'>" + prioOpts + "</select><input class='grow' id='nc-tags' type='text' placeholder='Tags, comma-separated' aria-label='Tags' value='" + esc(st.tags) + "'></div>";
  h += "<label style='display:flex;gap:8px;align-items:center;margin-top:10px;font-size:13px'><input type='checkbox' id='nc-sensitive' style='width:20px;height:20px'" + (st.sensitive ? " checked" : "") + "> Sensitive — its name and details stay home through the DHD door</label>";
  if (st.entities.length) {
    h += "<div class='mono' style='font-size:11px;color:var(--dim);margin-top:12px'>ENTITIES SHE SWIPED — UNVERIFIED LEADS</div>";
    st.entities.forEach(function (e, i) {
      h += "<label class='nc-check'><input type='checkbox' id='nc-ent-" + i + "'" + (e.on !== false ? " checked" : "") + "><span><b>" + esc(e.name) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(e.type.toUpperCase()) + "</span>" + (e.role ? "<br><span style='color:var(--dim);font-size:12px'>" + esc(e.role) + "</span>" : "") + "</span></label>";
    });
  }
  if (st.observables.length) {
    h += "<div class='mono' style='font-size:11px;color:var(--dim);margin-top:12px'>OBSERVABLES SHE SWIPED — UNVERIFIED LEADS</div>";
    st.observables.forEach(function (o, i) {
      h += "<label class='nc-check'><input type='checkbox' id='nc-obs-" + i + "'" + (o.on !== false ? " checked" : "") + "><span><b class='mono' style='font-size:13px'>" + esc(o.value) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(o.kind.toUpperCase()) + "</span>" + (o.context ? "<br><span style='color:var(--dim);font-size:12px'>" + esc(o.context) + "</span>" : "") + "</span></label>";
    });
  }
  h += "<div class='row' style='margin-top:14px'><button class='ghost' data-nc-cancel style='min-height:44px'>Cancel</button><button class='act grow' id='nc-submit' style='min-height:44px'" + (st.busy || st.submitting ? " disabled" : "") + ">" + (st.submitting ? "Starting…" : "Start the case") + "</button></div>";
  return h + "</div>";
}
function harvestNewCaseForm() {
  var st = newCaseState;
  if (!st) return;
  var g = function (id) { var el = document.getElementById(id); return el ? el.value : ""; };
  st.name = g("nc-name"); st.client = g("nc-client"); st.scope = g("nc-scope"); st.summary = g("nc-summary");
  st.type = g("nc-type"); st.priority = g("nc-priority"); st.tags = g("nc-tags");
  var sen = document.getElementById("nc-sensitive");
  st.sensitive = !!(sen && sen.checked);
  st.entities.forEach(function (e, i) { var c = document.getElementById("nc-ent-" + i); if (c) e.on = !!c.checked; });
  st.observables.forEach(function (o, i) { var c = document.getElementById("nc-obs-" + i); if (c) o.on = !!c.checked; });
}
function submitNewCaseForm() {
  var st = newCaseState;
  if (!st || st.submitting || st.busy) return;
  harvestNewCaseForm();
  if (!st.name.trim()) { st.note = "A case needs a name — give it one and it can be started."; render(); return; }
  st.submitting = true; st.note = ""; render();
  var fail = function (what) {
    if (!newCaseState) return;
    newCaseState.submitting = false;
    newCaseState.note = what;
    render();
  };
  ncPost("/api/cases/create", { name: st.name, client: st.client, scope: st.scope, fields: { summary: st.summary, type: st.type || "investigation", priority: st.priority || "normal", tags: st.tags, sensitive: !!st.sensitive } }, function (j) {
    if (!newCaseState) return;
    if (!j.ok || !j.case) { fail(j.error || "The case could not be opened."); return; }
    var slug = j.case.slug;
    var queue = [];
    st.entities.forEach(function (e) { if (e.on !== false) queue.push({ register: "entities", heading: e.name, fields: { type: e.type || "other", role: e.role || "" }, body: "" }); });
    st.observables.forEach(function (o) { if (o.on !== false) queue.push({ register: "observables", heading: o.value, fields: { kind: o.kind || "other" }, body: o.context || "" }); });
    var step = function (i) {
      if (!newCaseState) return;
      if (i >= queue.length) {
        ncPost("/api/cases/open", { case: slug }, function () {
          newCaseState = null;
          caseOpenSlug = slug;
          try { localStorage.setItem("navi.openCase", slug); } catch (e) {}
          hydrateChatsFromServer();
          jumpToCaseChat(slug, function () { render(); });
        }, function () { newCaseState = null; jumpToCaseChat(slug, function () { render(); }); });
        return;
      }
      var q = queue[i];
      ncPost("/api/cases/register", { case: slug, register: q.register, heading: q.heading, fields: q.fields, body: q.body, unverified: true }, function (rj) {
        if (!newCaseState) return;
        if (!rj.ok) { fail((rj.error || "An entry could not be filed.") + " The case itself stands — find it in Case Files, and the rest can be filed from there."); return; }
        step(i + 1);
      }, function () { fail("The bridge stopped answering mid-filing. The case itself stands — find it in Case Files, and the rest can be filed from there."); });
    };
    step(0);
  }, function () { fail("The bridge didn't answer — nothing was created."); });
}
function afterCaseDeleted(slug, name) {
  // After a case delete lands (Build 15, #163; shared by the
  // detail card and, since Build 3's #177, the cases list): the
  // tied chat goes with the case — out of this window's store
  // too, so the write-through cannot resurrect it at the next
  // persist — the open-case pointer clears if it pointed here,
  // and the view resets to the list with the verdict stated.
  var before = chats.length;
  chats = chats.filter(function (c) { return c.case !== slug; });
  if (!chats.length) seedChats();
  if (!chats.some(function (c) { return c.id === activeChatId; })) { activeChatId = chats[0].id; messages = chats[0].messages; }
  if (caseOpenSlug === slug) { caseOpenSlug = ""; try { localStorage.setItem("navi.openCase", ""); } catch (e) {} openCaseState = null; }
  caseDetailState = null; casesState = null; caseReportState = null;
  caseMsg = "Deleted — " + name + " and all of its files are gone from the Vault" + (chats.length !== before ? ", and its chat went with it" : "") + ". The deletion travels with Sync, so it won't come back from your other machine. Copies inside backups already taken are untouched.";
  persist();
}
function wireCaseFiles() {
  var bind = function (id, fn) { var el = document.getElementById(id); if (el) el.onclick = fn; };
  bind("cf-create", function () {
    var name = (document.getElementById("cf-name") || {}).value || "";
    var client = (document.getElementById("cf-client") || {}).value || "";
    var scope = (document.getElementById("cf-scope") || {}).value || "";
    var fields = {
      summary: (document.getElementById("cf-summary") || {}).value || "",
      type: (document.getElementById("cf-type") || {}).value || "investigation",
      priority: (document.getElementById("cf-priority") || {}).value || "normal",
      tags: (document.getElementById("cf-tags") || {}).value || "",
      sensitive: !!((document.getElementById("cf-sensitive") || {}).checked),
    };
    casePost("/api/cases/create", { name: name, client: client, scope: scope, fields: fields }, function (j) {
      if (j.ok) { caseMsg = "Case opened — it lives at cases/" + j.case.slug + "/ on the desk, with its own chat on the Chats tab."; casesState = null; caseDetailState = j.case; hydrateChatsFromServer(); }
      else caseMsg = j.error || "The case could not be opened.";
      render();
    });
  });
  // Build 2 (#199): the sensitive switch — the switch gate's
  // three-way contract: the flip posts to the bridge, the
  // detail re-reads the STORED value, and the re-render shows
  // what the bridge holds. A refusal lands as a message on
  // this same card and the checkbox re-renders from the
  // stored state — never a silent snap-back.
  var sensToggle = document.getElementById("cf-sensitive-toggle");
  if (sensToggle && caseDetailState) sensToggle.onchange = function () {
    var slug = caseDetailState.slug;
    var want = !!sensToggle.checked;
    casePost("/api/cases/update", { case: slug, fields: { sensitive: want ? "yes" : "no" } }, function (j) {
      if (j.ok && j.case) {
        caseDetailState = j.case; casesState = null;
        caseEditMsg = j.case.sensitive === want
          ? (want ? "Sensitive is on — the bridge stored it; the DHD list now redacts this case and Sync badges it." : "Sensitive is off — the bridge stored it.")
          : "The bridge stored a different value than the flip asked for — showing what it holds.";
      } else {
        caseEditMsg = "The sensitive change could not be saved: " + (j.error || "the bridge didn't answer") + " — the switch shows the stored state.";
      }
      render();
    });
  };
  bind("cf-edit-toggle", function () { caseEditOpen = !caseEditOpen; caseEditMsg = ""; render(); });
  bind("cf-edit-save", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    var fields = {
      summary: (document.getElementById("cf-e-summary") || {}).value || "",
      type: (document.getElementById("cf-e-type") || {}).value || "investigation",
      priority: (document.getElementById("cf-e-priority") || {}).value || "normal",
      status: (document.getElementById("cf-e-status") || {}).value || "open",
      tags: (document.getElementById("cf-e-tags") || {}).value || "",
      owner: (document.getElementById("cf-e-owner") || {}).value || "",
      related: (document.getElementById("cf-e-related") || {}).value || "",
      outcome: (document.getElementById("cf-e-outcome") || {}).value || "",
    };
    casePost("/api/cases/update", { case: slug, fields: fields }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; casesState = null; caseEditOpen = false; caseEditMsg = "Saved — the case shows what the bridge stored."; }
      else caseEditMsg = "The case could not be saved: " + (j.error || "the bridge didn't answer");
      render();
    });
  });
  // The finding delete door (v0.5.9e, #280): arm, keep,
  // fire — the fire posts to the bridge's case door, which
  // routes element findings to the global delete.
  mainEl.querySelectorAll("[data-cf-delfinding]").forEach(function (b) { b.onclick = function () { cfDelArm = b.getAttribute("data-cf-delfinding"); render(); }; });
  mainEl.querySelectorAll("[data-cf-delfinding-no]").forEach(function (b) { b.onclick = function () { cfDelArm = ""; render(); }; });
  mainEl.querySelectorAll("[data-cf-delfinding-ok]").forEach(function (b) { b.onclick = function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    var r = document.getElementById("cf-del-reason");
    cfDelArm = "";
    casePost("/api/cases/delete-finding", { case: slug, file: b.getAttribute("data-cf-delfinding-ok"), reason: r ? r.value.trim() : "" }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; caseMsg = "Deleted — the case's custody record carries the line."; }
      else caseMsg = j.error || "The finding could not be deleted.";
      render();
    });
  }; });
  // Build 2 (#199): one-tap review + register adds.
  mainEl.querySelectorAll("[data-cf-review]").forEach(function (b) { b.onclick = function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    casePost("/api/cases/review", { case: slug, register: b.getAttribute("data-cf-review"), key: b.getAttribute("data-cf-key"), action: b.getAttribute("data-cf-action") }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; caseMsg = "Review recorded."; }
      else caseMsg = j.error || "The review could not be recorded.";
      render();
    });
  }; });
  bind("cf-ent-add", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    casePost("/api/cases/register", { case: slug, register: "entities", heading: (document.getElementById("cf-ent-name") || {}).value || "", fields: { type: (document.getElementById("cf-ent-type") || {}).value || "other", role: (document.getElementById("cf-ent-role") || {}).value || "" }, body: "" }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; caseMsg = "Entity added — confirmed, as your own filing."; }
      else caseMsg = j.error || "The entity could not be added.";
      render();
    });
  });
  bind("cf-obs-add", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    casePost("/api/cases/register", { case: slug, register: "observables", heading: (document.getElementById("cf-obs-value") || {}).value || "", fields: { kind: (document.getElementById("cf-obs-kind") || {}).value || "other" }, body: (document.getElementById("cf-obs-ctx") || {}).value || "" }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; caseMsg = "Observable added — it now feeds the Wire's case matching."; }
      else caseMsg = j.error || "The observable could not be added.";
      render();
    });
  });
  bind("cf-task-add", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    casePost("/api/cases/register", { case: slug, register: "tasks", heading: (document.getElementById("cf-task-text") || {}).value || "", fields: {}, body: "" }, function (j) {
      if (j.ok && j.case) { caseDetailState = j.case; caseMsg = "Task added."; }
      else caseMsg = j.error || "The task could not be added.";
      render();
    });
  });
  bind("cf-back", function () { caseDetailState = null; casesState = null; caseReportState = null; caseDeleteArm = ""; caseListDeleteArm = ""; render(); });
  mainEl.querySelectorAll("[data-case-open]").forEach(function (b) { b.onclick = function () { caseReportState = null; caseDeleteArm = ""; caseListDeleteArm = ""; refreshCaseDetail(b.getAttribute("data-case-open")); }; });
  mainEl.querySelectorAll("[data-case-room]").forEach(function (b) { b.onclick = function () { var parts = (b.getAttribute("data-case-room") || "").split("|"); fcFed = parts[0]; fcRoomId = parts[1] || ""; goView("FediChat"); refreshFedichat(false); }; });
  bind("cf-setopen", function () {
    if (!caseDetailState) return;
    var slug = caseOpenSlug === caseDetailState.slug ? "" : caseDetailState.slug;
    caseOpenSlug = slug;
    try { localStorage.setItem("navi.openCase", slug); } catch (e) {}
    casePost("/api/cases/open", { case: slug }, function () { caseMsg = slug ? "The open case is now " + caseDetailState.name + " — Navi files to it from chat." : "No case is open now — chat filings will say so."; if (slug) hydrateChatsFromServer(); render(); });
  });
  bind("cf-chat", function () {
    if (!caseDetailState) return;
    jumpToCaseChat(caseDetailState.slug, function (j) {
      caseMsg = j.error || "The case chat could not be opened.";
      render();
    });
  });
  bind("cf-del", function () { if (caseDetailState) { caseDeleteArm = caseDetailState.slug; render(); } });
  bind("cf-del-no", function () { caseDeleteArm = ""; render(); });
  bind("cf-del-yes", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug, name = caseDetailState.name;
    casePost("/api/cases/delete", { case: slug }, function (j) {
      caseDeleteArm = "";
      if (j.ok) afterCaseDeleted(slug, name);
      else caseMsg = j.error || "The case could not be deleted.";
      render();
    });
  });
  // The cases LIST's delete (Build 3, #177): same endpoint, same
  // cleanup — the confirm lives in the card itself, naming the
  // case with the counts the list already carries.
  mainEl.querySelectorAll("[data-case-del]").forEach(function (b) { b.onclick = function () { caseListDeleteArm = b.getAttribute("data-case-del"); render(); }; });
  mainEl.querySelectorAll("[data-case-del-no]").forEach(function (b) { b.onclick = function () { caseListDeleteArm = ""; render(); }; });
  mainEl.querySelectorAll("[data-case-del-yes]").forEach(function (b) { b.onclick = function () {
    var slug = b.getAttribute("data-case-del-yes"), name = b.getAttribute("data-case-name") || slug;
    casePost("/api/cases/delete", { case: slug }, function (j) {
      caseListDeleteArm = "";
      if (j.ok) afterCaseDeleted(slug, name);
      else caseMsg = j.error || "The case could not be deleted.";
      render();
    });
  }; });
  bind("cf-f-add", function () {
    if (!caseDetailState) return;
    var title = (document.getElementById("cf-f-title") || {}).value || "";
    var body = (document.getElementById("cf-f-body") || {}).value || "";
    var sev = (document.getElementById("cf-f-sev") || {}).value || "info";
    var internal = !!((document.getElementById("cf-f-internal") || {}).checked);
    casePost("/api/cases/file", { case: caseDetailState.slug, kind: "finding", title: title, body: body, severity: sev, internal: internal }, function (j) {
      caseMsg = j.ok ? "Finding filed at " + j.filed.path + "." : (j.error || "The finding could not be filed.");
      refreshCaseDetail(caseDetailState.slug); render();
    });
  });
  bind("cf-file-exec", function () {
    if (!caseDetailState || !lastChatExec) return;
    var title = "Chat output — " + String(lastChatExec.command || "chip run").slice(0, 60);
    var body = "```\n$ " + (lastChatExec.command || "") + "\n" + (lastChatExec.output || "") + "\n```";
    casePost("/api/cases/file", { case: caseDetailState.slug, kind: "finding", title: title, body: body, severity: "info", internal: false }, function (j) {
      caseMsg = j.ok ? "The chat output is filed as a finding at " + j.filed.path + "." : (j.error || "It could not be filed.");
      refreshCaseDetail(caseDetailState.slug); render();
    });
  });
  bind("cf-t-add", function () {
    if (!caseDetailState) return;
    var title = (document.getElementById("cf-t-title") || {}).value || "";
    var body = (document.getElementById("cf-t-body") || {}).value || "";
    casePost("/api/cases/file", { case: caseDetailState.slug, kind: "timeline", title: title, body: body }, function (j) {
      caseMsg = j.ok ? "Timeline entry added." : (j.error || "The entry could not be added.");
      refreshCaseDetail(caseDetailState.slug); render();
    });
  });
  bind("cf-report", function () {
    if (!caseDetailState) return;
    var slug = caseDetailState.slug;
    casePost("/api/cases/report", { case: slug }, function (j) {
      if (j.ok) { caseReportState = { slug: slug, text: j.report, withheld: j.withheld || 0 }; caseMsg = ""; }
      else caseMsg = j.error || "The report could not be composed.";
      render();
    });
  });
  bind("cf-report-dl", function () {
    if (!caseReportState) return;
    var blob = new Blob([caseReportState.text], { type: "text/markdown" });
    var a = document.createElement("a");
    a.href = URL.createObjectURL(blob); a.download = "report.md"; a.click();
    setTimeout(function () { try { URL.revokeObjectURL(a.href); } catch (e) {} }, 4000);
  });
  var drop = document.getElementById("cf-drop");
  var fileInput = document.getElementById("cf-files");
  if (drop && fileInput) {
    drop.onclick = function () { fileInput.click(); };
    drop.ondragover = function (ev) { ev.preventDefault(); if (!caseDragOver) { caseDragOver = true; render(); } };
    drop.ondragleave = function () { caseDragOver = false; render(); };
    drop.ondrop = function (ev) { ev.preventDefault(); caseDragOver = false; caseEvidenceFiles(ev.dataTransfer ? ev.dataTransfer.files : null); };
    fileInput.onchange = function () { caseEvidenceFiles(fileInput.files); };
  }
}
function caseEvidenceFiles(files) {
  if (!caseDetailState || !files || !files.length) { render(); return; }
  var slug = caseDetailState.slug;
  var list = Array.prototype.slice.call(files);
  var done = 0;
  list.forEach(function (f) {
    var reader = new FileReader();
    reader.onload = function () {
      var url = String(reader.result || "");
      var b64 = url.indexOf(",") !== -1 ? url.slice(url.indexOf(",") + 1) : "";
      casePost("/api/cases/evidence", { case: slug, name: f.name, dataBase64: b64 }, function (j) {
        done++;
        if (done === list.length) { caseMsg = "Evidence filed: " + list.map(function (x) { return x.name; }).join(", ") + "."; refreshCaseDetail(slug); render(); }
      });
    };
    reader.onerror = function () { done++; if (done === list.length) render(); };
    reader.readAsDataURL(f);
  });
}

/* --------------------------- command lists ----------------------------- */
// Autocomplete draws from what the system has installed: the built-in
// default set, plus installed commands. In live mode the installed list
// is bridge.py's real PATH scan; in mock mode it is managed in Settings.
var DEFAULT_COMMANDS = [
  { name: "ls", description: "List directory contents" },
  { name: "cat", description: "Print a file's contents" },
  { name: "cd", description: "Change directory" },
  { name: "pwd", description: "Print the working directory" },
  { name: "echo", description: "Print text back" },
  { name: "whoami", description: "Print the current user" },
  { name: "hostname", description: "Print the host name" },
  { name: "date", description: "Print the current date and time" },
  { name: "uname", description: "Print system information" },
  { name: "ps", description: "List running processes" },
  { name: "navi", description: "Navi system status" },
  { name: "help", description: "List the shell's commands" }
];
var CATALOG = [
  { name: "nmap", description: "Network discovery and port scanning" },
  { name: "grep", description: "Search text with patterns" },
  { name: "find", description: "Find files by name or attribute" },
  { name: "curl", description: "Transfer data from a URL" },
  { name: "ssh", description: "Remote shell to another host" },
  { name: "ping", description: "Check whether a host answers" },
  { name: "tail", description: "Print the end of a file" },
  { name: "dig", description: "DNS lookups" },
  { name: "netstat", description: "Show network connections and sockets" },
  { name: "whois", description: "Look up domain registration" }
];
var liveCommands = null;   // names from /api/commands (live mode)
function getCommands() {
  var out = DEFAULT_COMMANDS.map(function (c) { return { name: c.name, description: c.description, source: "default" }; });
  if (CONFIG.backend === "live" && liveCommands) {
    liveCommands.forEach(function (n) {
      if (!DEFAULT_COMMANDS.some(function (c) { return c.name === n; })) out.push({ name: n, description: "Installed on this system", source: "installed" });
    });
  } else {
    (CONFIG.installed || []).forEach(function (n) {
      var cat = CATALOG.filter(function (c) { return c.name === n; })[0];
      out.push({ name: n, description: cat ? cat.description : "Installed by the operator", source: "installed" });
    });
  }
  return out;
}
function ensureLiveCommands() {
  if (CONFIG.backend !== "live" || liveCommands) return;
  fetch("/api/commands").then(function (r) { return r.json(); }).then(function (j) { liveCommands = j.commands || []; }).catch(function () { liveCommands = []; });
}

/* ------------------ CLI-first routing + line coloring ----------------- */
// The dashboard mirror of the bridge's cli_route: "$" forces the
// shell, "> " forces Navi, the composer's latched mode (its CLI /
// Navi press-once toggles) decides unmarked lines, and only with
// no latch passed does a first word the system knows — the same
// getCommands() universe autocomplete offers — make a command.
// The composer predicts the route for its mode hint and live
// styling; the bridge makes the real call in live mode.
function routeOfLine(raw, latchedMode) {
  var text = String(raw || "").trim();
  if (text.charAt(0) === "$") return { kind: "cli", command: text.slice(1).trim(), forced: true };
  if (text === ">" || text.indexOf("> ") === 0) return { kind: "navi", text: text.slice(1).trim(), forced: true };
  if (latchedMode === "cli") return { kind: "cli", command: text, forced: false };
  if (latchedMode === "navi") return { kind: "navi", text: text, forced: false };
  var body = text;
  if (body === "sudo") body = "";
  else if (body.indexOf("sudo ") === 0) body = body.slice(5).trim();
  var first = (body.split(/\s+/)[0] || "").toLowerCase();
  if (first.indexOf("/") !== -1) first = first.split("/").pop();
  if (first && getCommands().some(function (c) { return c.name === first; })) return { kind: "cli", command: text, forced: false };
  return { kind: "navi", text: text, forced: false };
}
// The zsh-syntax-highlighting read of a command line, as HTML: the
// command word colored by whether the system knows it, sudo a
// keyword, flags, quoted strings, and paths each their own color.
// Used by the composer's live overlay and by the thread's terminal
// blocks, so a command reads the same while typed and once echoed.
function cmdTokensHtml(line) {
  var known = function (n) { return n === "chip" || getCommands().some(function (c) { return c.name === n; }); };
  var out = "", i = 0, expect = true;
  while (i < line.length) {
    var ch = line.charAt(i);
    if (/\s/.test(ch)) { var j = i; while (j < line.length && /\s/.test(line.charAt(j))) j++; out += esc(line.slice(i, j)); i = j; continue; }
    if (ch === '"' || ch === "'") { var k = i + 1; while (k < line.length && line.charAt(k) !== ch) k++; if (k < line.length) k++; out += "<span class='tk-str'>" + esc(line.slice(i, k)) + "</span>"; expect = false; i = k; continue; }
    if (ch === "$" && line.charAt(i + 1) === "{") { var b = line.indexOf("}", i + 2); var vend = b === -1 ? line.length : b + 1; out += "<span class='tk-kw'>" + esc(line.slice(i, vend)) + "</span>"; expect = false; i = vend; continue; }
    var two = line.slice(i, i + 2);
    if (two === ">>" || two === "&&" || two === "||") { out += "<span class='tk-op'>" + esc(two) + "</span>"; expect = true; i += 2; continue; }
    if (ch === "|" || ch === ">" || ch === "<" || ch === ";") { out += "<span class='tk-op'>" + esc(ch) + "</span>"; expect = true; i++; continue; }
    var e = i; while (e < line.length && !/\s/.test(line.charAt(e)) && "|><;".indexOf(line.charAt(e)) === -1 && !(line.charAt(e) === "&" && line.charAt(e + 1) === "&")) e++;
    if (e === i) { out += esc(ch); i++; continue; }
    var w = line.slice(i, e); i = e;
    if (w.charAt(0) === "$" && /^\$[A-Za-z_][A-Za-z0-9_]*$/.test(w)) { out += "<span class='tk-kw'>" + esc(w) + "</span>"; expect = false; continue; }
    if (/^-?\d+(\.\d+)?$/.test(w)) { out += "<span class='tk-num'>" + esc(w) + "</span>"; expect = false; continue; }
    if (expect && (w === "sudo" || w === "$")) { out += "<span class='tk-kw'>" + esc(w) + "</span>"; continue; }
    if (expect) { expect = false; var base = w.indexOf("/") !== -1 ? w.split("/").pop() : w; out += "<span class='" + (known(String(base).toLowerCase()) ? "tk-cmd" : "tk-bad") + "'>" + esc(w) + "</span>"; continue; }
    if (w.length > 1 && w.charAt(0) === "-") out += "<span class='tk-flag'>" + esc(w) + "</span>";
    else if (w.charAt(0) === "~" || w.indexOf("/") !== -1) out += "<span class='tk-path'>" + esc(w) + "</span>";
    else out += esc(w);
  }
  return out;
}
function execHtml(x) {
  return "<div class='exec mono'><span style='color:var(--cy)'>$ </span>" + cmdTokensHtml(x.command) + "\n" + esc(x.output) + "</div>";
}
function execExitInfo(x) {
  if (x && typeof x.exit_code === "number") return { code: x.exit_code, label: "exit " + x.exit_code, known: true };
  var m = String((x && x.output) || "").match(/\[exit (\d+)\]\s*$/);
  if (m) return { code: Number(m[1]), label: "exit " + m[1], known: true };
  return { code: null, label: "finished", known: false };
}
function execOutputClean(x) {
  var out = String((x && x.output) || "");
  if (x && typeof x.exit_code !== "number" && /\[exit \d+\]\s*$/.test(out)) out = out.replace(/\n?\[exit \d+\]\s*$/, "");
  return out;
}
function execChatHtml(x) {
  var info = execExitInfo(x);
  var h = "<div class='exec mono'>" + esc(execOutputClean(x)) + "</div>";
  if (info.known) h += "<div class='execstat mono'>" + esc(info.label) + "</div>";
  return h;
}
/* CLI Mode shell sessions (Build 3, #209): the page side. The
   session itself lives in the bridge; this is its record and its
   controls. Events poll in on a light timer while the Chat view
   is open, fold into the thread as SHELL records (persisted with
   the chat like exec blocks), and a full-screen program renders
   on the grid the bridge keeps \u2014 desktop pane only; a phone
   gets the streamed record and a note naming the difference. */
var shellUi = { chatId: null, live: false, ended: false, altScreen: false, screen: null, seq: 0, timer: null, fetching: false };
function shellEventsHtml(events) {
  var h = "<div class='shellrec mono'>";
  (events || []).forEach(function (e) {
    if (e.kind === "in") h += "<div class='shell-in'><span class='shell-badge " + (e.author === "navi" ? "navi" : "op") + "'>" + (e.author === "navi" ? "NAVI" : "OPERATOR") + "</span> " + esc(e.text) + "</div>";
    else if (e.kind === "out") h += "<pre class='shell-out'>" + esc(e.text) + "</pre>";
    else h += "<div class='shell-note'>" + esc(e.text) + "</div>";
  });
  return h + "</div>";
}
function shellFoldEvents(events, endedNow) {
  if (!events.length && !endedNow) return;
  var last = messages[messages.length - 1];
  if (!last || last.who !== "SHELL" || !last.shell || !last.shell.open) {
    last = { who: "SHELL", text: "", shell: { events: [], open: true }, ts: Date.now() };
    messages.push(last);
  }
  events.forEach(function (e) { last.shell.events.push(e); });
  if (endedNow) last.shell.open = false;
  persist();
  var msgsEl = document.getElementById("msgs");
  if (msgsEl) {
    var kids = msgsEl.querySelectorAll(".msg");
    var lastMsgEl = kids.length ? kids[kids.length - 1] : null;
    var rec = lastMsgEl ? lastMsgEl.querySelector(".shellrec") : null;
    if (rec && messages[messages.length - 1] === last && lastMsgEl.dataset.shellrec === "1") {
      rec.outerHTML = shellEventsHtml(last.shell.events);
    } else {
      var holder = document.createElement("div");
      holder.innerHTML = "<div class='msg' data-shellrec='1'><div class='msg-row'>" + msgAvatarHtml(last) + "<div class='msg-body'><div class='who'>" + msgWhoHtml(last) + "</div>" + shellEventsHtml(last.shell.events) + "</div></div></div>";
      msgsEl.appendChild(holder.firstChild);
    }
    msgsEl.scrollTop = msgsEl.scrollHeight;
  }
}
function shellPoll() {
  // Returns the poll's promise (resolving once the state has
  // folded in) so callers that act on the session's surface —
  // the open, the bring-forward — can wait for the fold
  // instead of racing it. Fire-and-forget callers (the 900 ms
  // watch) simply ignore the return.
  if (view !== "Chat" || CONFIG.backend !== "live" || shellUi.fetching) return Promise.resolve();
  var cid = String(activeChatId);
  if (shellUi.chatId !== cid) { shellUi.chatId = cid; shellUi.seq = 0; shellUi.live = false; shellUi.ended = false; shellUi.altScreen = false; shellUi.screen = null; }
  shellUi.fetching = true;
  return fetch("/api/shell/session/state?chatId=" + encodeURIComponent(cid) + "&since=" + shellUi.seq)
    .then(function (r) { return r.json(); })
    .then(function (st) {
      shellUi.fetching = false;
      if (!st || view !== "Chat" || String(activeChatId) !== cid) return;
      var hadLive = shellUi.live;
      var wasAlt = shellUi.altScreen;
      var wasLive = shellUi.live;
      shellUi.live = !!st.live; shellUi.ended = !!st.ended;
      shellUi.altScreen = !!st.altScreen; shellUi.screen = st.screen || null;
      if (typeof st.seq === "number") shellUi.seq = st.seq;
      if (st.events && st.events.length) shellFoldEvents(st.events, shellUi.ended);
      if (hadLive && !st.live && !st.ended) {
        shellFoldEvents([{ kind: "note", author: null, text: "the session ended when the bridge restarted \u2014 the shell is gone; nothing was carried over" }], true);
        shellUi.ended = true;
      }
      if (wasAlt !== shellUi.altScreen || wasLive !== shellUi.live) { render(); return; }
      if (shellUi.altScreen && shellUi.screen) {
        var pre = document.getElementById("shell-screen");
        if (pre) pre.textContent = shellUi.screen.join("\n");
      }
    })
    .catch(function () { shellUi.fetching = false; });
}
function watchShellSession() {
  if (shellUi.timer) return;
  shellUi.timer = setInterval(shellPoll, 900);
}
function shellPost(path, body) {
  return fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) }).then(function (r) { return r.json(); });
}
function shellSessionOpen() {
  // Open (or return) this chat's standing session. Resolves
  // with the bridge's state payload; REJECTS with the refusal's
  // own words when the bridge will not open one, so callers can
  // degrade honestly instead of swallowing the failure.
  var cols = Math.max(40, Math.min(200, Math.floor(window.innerWidth / 8)));
  return shellPost("/api/shell/session/open", { chatId: String(activeChatId), cols: cols, rows: 24 }).then(function (j) {
    if (j && j.ok === false) throw new Error(j.error || "the bridge refused the session");
    shellUi.seq = 0; shellUi.chatId = String(activeChatId);
    // The open completes only once the session's state has
    // folded into the thread — callers that bring the session
    // forward (Build 5, #219) scroll to a record that exists.
    return shellPoll().then(function () { render(); return j; });
  });
}
function shellNoteInThread(text) {
  // A session-shaped note in the thread: the refusal, named in
  // the terminal record's own voice, never a silent failure.
  messages.push({ who: "SHELL", text: "", shell: { events: [{ kind: "note", author: null, text: text }], open: false }, ts: Date.now() });
  persist(); render();
}
var shellOpenInFlight = false;
function shellBringForward() {
  // Build 5 (#219): bringing the session forward is an ACT,
  // not a state check. Build 4's resume path trusted the
  // poll-fed shellUi.live cache and, when it read live, did
  // NOTHING \u2014 no render, no scroll, no focus \u2014 so a
  // session whose surface had drifted out of view (or whose
  // cached flag had drifted from the bridge) stayed
  // invisible, and the flip that should have restored it was
  // a no-op. Now every CLI latch re-asserts the surface: a
  // fresh state poll (the cache is never the last word on a
  // flip), a render, the thread scrolled to the terminal
  // record, and the composer focused \u2014 the session's
  // input, now that the composer is its only bar.
  watchShellSession();
  shellPoll();
  render();
  var msgsEl = document.getElementById("msgs");
  if (msgsEl) {
    var recs = msgsEl.querySelectorAll("[data-shellrec]");
    if (recs.length) { try { recs[recs.length - 1].scrollIntoView({ block: "end" }); } catch (e) { msgsEl.scrollTop = msgsEl.scrollHeight; } }
    else msgsEl.scrollTop = msgsEl.scrollHeight;
  }
  var inp = document.getElementById("chat-in");
  if (inp) inp.focus();
}
function chatModeToggled(m) {
  // Build 4 (#214): the composer's CLI latch IS CLI Mode now.
  // Flipping to CLI opens (or resumes) this chat's standing
  // session; flipping back to Navi just routes input to chat
  // again \u2014 the session persists behind the toggle under
  // its own close rules. The switch gate: if the bridge
  // refuses the open, the flip degrades honestly \u2014 the latch
  // reverts to Navi and the refusal is named in the thread.
  // Build 5 (#219): a resume brings the session FORWARD \u2014
  // shellBringForward() runs on every CLI latch, whether the
  // session was just opened or was alive behind the toggle.
  if (m !== "cli") return;
  if (CONFIG.backend !== "live" || view !== "Chat") return;
  if (shellUi.live && shellUi.chatId === String(activeChatId)) { shellBringForward(); return; }
  if (shellOpenInFlight) return;
  shellOpenInFlight = true;
  shellSessionOpen().then(function () { shellOpenInFlight = false; shellBringForward(); }).catch(function (e) {
    shellOpenInFlight = false;
    chatMode = "navi"; persistChatMode();
    shellNoteInThread("The shell session could not be opened \u2014 " + (e && e.message ? e.message : "the bridge could not be reached") + ". The mode stayed with Navi.");
  });
}
function sendLineToShellSession(text) {
  // Build 4 (#214): a CLI-latched line is typed into the chat's
  // standing session \u2014 the terminal record is its rendering,
  // its input badged OPERATOR by the bridge's own event fold.
  // A leading $ is the latch gesture, not part of the line.
  // If the session cannot take the line, the failure is named
  // in the thread and the line runs NOWHERE else \u2014 never a
  // silent fallback to a one-shot or to Navi.
  var line = String(text || "").replace(/^\$\s*/, "");
  try { cliHistoryAppend(line); } catch (e) {}
  var deliver = function () {
    shellPost("/api/shell/session/input", { chatId: String(activeChatId), text: line + "\n" }).then(function (j) {
      if (j && j.ok === false) shellNoteInThread("The line was not typed into the session \u2014 " + (j.error || "no live session") + " It was not run anywhere else.");
      shellPoll();
    }).catch(function () {
      shellNoteInThread("The line could not reach the shell session \u2014 the bridge could not be reached. It was not run anywhere else.");
    });
  };
  if (shellUi.live && shellUi.chatId === String(activeChatId)) deliver();
  else shellSessionOpen().then(deliver).catch(function (e) {
    shellNoteInThread("The shell session could not be opened \u2014 " + (e && e.message ? e.message : "the bridge could not be reached") + ". The line was not run.");
  });
}
function wireShellUi() {
  // Build 5 (#219): the session's only control is Close, and
  // it lives in the session header / the pane head — both
  // carry data-shell-close and both bind here. (The retired
  // bar's own input and Type button are gone: the composer is
  // the session's input, wired with the rest of the composer.)
  document.querySelectorAll("[data-shell-close]").forEach(function (close) {
    close.onclick = function () {
      shellPost("/api/shell/session/close", { chatId: String(activeChatId) }).then(function () { shellPoll(); }).catch(function () {});
    };
  });
  var pane = document.getElementById("shell-pane");
  if (pane) pane.onkeydown = function (e) {
    var k = null;
    if (e.key === "Enter") k = "\r";
    else if (e.key === "Backspace") k = "\x7f";
    else if (e.key === "Tab") k = "\t";
    else if (e.key === "Escape") k = "\x1b";
    else if (e.key === "ArrowUp") k = "\x1b[A";
    else if (e.key === "ArrowDown") k = "\x1b[B";
    else if (e.key === "ArrowRight") k = "\x1b[C";
    else if (e.key === "ArrowLeft") k = "\x1b[D";
    else if (e.ctrlKey && e.key.length === 1) k = String.fromCharCode(e.key.toUpperCase().charCodeAt(0) - 64);
    else if (e.key.length === 1) k = e.key;
    if (k !== null) { e.preventDefault(); shellPost("/api/shell/session/input", { chatId: String(activeChatId), text: k, raw: true }).then(function () { shellPoll(); }).catch(function () {}); }
  };
}
function shellPaneVisible() {
  // The desktop full-screen pane: live, a program holding the
  // alternate screen, and a screen wide enough that the pane
  // renders (the phone block hides it by CSS — there the
  // header below carries the session instead).
  return !!(shellUi.live && shellUi.altScreen && shellUi.screen && window.innerWidth > 800);
}
function shellHeadHtml() {
  // Build 5 (#219): the session's header — status and Close,
  // NO input. The Build 3 session bar duplicated the composer
  // once Build 4 made the composer the session's input, so
  // the bar retired; what it carried that the composer does
  // not — the live status and the Close — lives here, and in
  // the pane head when the pane is the session's face.
  if (CONFIG.backend !== "live" || !shellUi.live) return "";
  if (shellPaneVisible()) return "";
  return "<div class='shellhead' id='shell-head' data-live='1'><span class='dot' style='background:#34d399'></span><span style='font-size:12px'><b>Shell session</b> \u2014 live in this chat" + (shellUi.altScreen ? " \u00b7 a program holds the full screen" : "") + "</span>" +
    "<button class='ghost shell-close' data-shell-close>Close session</button>" +
    (shellUi.altScreen ? "<span class='shell-mobile-note'>A full-screen program is running. On this screen size Apex streams its output into the thread instead of rendering the terminal pane \u2014 the pane is a desktop surface.</span>" : "") +
    "</div>";
}
function shellPaneHtml() {
  if (!shellUi.live || !shellUi.altScreen || !shellUi.screen) return "";
  return "<div class='shellpane' id='shell-pane' tabindex='0' aria-label='Terminal pane \u2014 click and type to the full-screen program'><div class='shellpane-head'><span>TERMINAL \u2014 FULL-SCREEN PROGRAM (CLICK AND TYPE)</span><span style='letter-spacing:0'><span class='dot' style='background:#34d399'></span> <b>Shell session</b> \u2014 live in this chat</span><button class='ghost shell-close' data-shell-close>Close session</button></div><pre id='shell-screen'>" + esc(shellUi.screen.join("\n")) + "</pre></div>";
}
function isLongNavi(text) {
  var t = String(text || "");
  return t.length >= 500 || t.split("\n").length >= 8 || t.indexOf("\x60\x60\x60") !== -1;
}
function firstHintLine(text, stripMd) {
  var lines = String(text || "").split("\n");
  for (var i = 0; i < lines.length; i++) { var l = lines[i].trim(); if (stripMd) l = l.replace(/^[#>*\-\x60\s]+/, "").replace(/[*_\x60]/g, "").trim(); if (l) return l.length > 90 ? l.slice(0, 90) + "…" : l; }
  return "";
}
var chatFoldState = {};
function looksLikeCommandLine(v) {
  var t = String(v || "").trim();
  if (!t) return false;
  if (t.charAt(0) === "$") return true;
  var first = (t.split(/\s+/)[0] || "").toLowerCase();
  if (first.indexOf("/") !== -1) first = first.split("/").pop();
  return getCommands().some(function (c) { return c.name === first; });
}
/* Persistent CLI history store (#27a): localStorage navi.cliHistory, chronological, cap 200. */
function cliHistoryStore() {
  try { var raw = localStorage.getItem("navi.cliHistory"); if (raw !== null) { var a = JSON.parse(raw); return Array.isArray(a) ? a.filter(function (x) { return typeof x === "string"; }) : []; } } catch (e) { return null; }
  try {
    var seed = [];
    (typeof chats !== "undefined" ? chats : []).forEach(function (c) { (c.messages || []).forEach(function (m) { if (m.exec && m.exec.command) seed.push(m.exec.command); else if (m.who === "YOU" && m.text) { try { if (routeOfLine(m.text).kind === "cli") seed.push(String(m.text).replace(/^\$\s*/, "")); } catch (e2) {} } }); });
    (typeof messages !== "undefined" ? messages : []).forEach(function (m) { if (m.exec && m.exec.command) seed.push(m.exec.command); });
    seed = seed.slice(-200);
    try { localStorage.setItem("navi.cliHistory", JSON.stringify(seed)); } catch (e3) {}
    return seed;
  } catch (e4) { return []; }
}
function cliHistoryAppend(line) {
  var v = String(line || "").trim(); if (!v) return;
  var a = cliHistoryStore(); if (!a) return;
  a.push(v); if (a.length > 200) a = a.slice(a.length - 200);
  try { localStorage.setItem("navi.cliHistory", JSON.stringify(a)); } catch (e) {}
}
function expandHistoryRefs(line, history) {
  var hist = history || [];
  var prev = hist.length ? hist[hist.length - 1] : null;
  var out = "", i = 0, changed = false, inS = false;
  while (i < line.length) {
    var ch = line.charAt(i);
    if (ch === "'") { inS = !inS; out += ch; i++; continue; }
    if (!inS && ch === "!") {
      var rest = line.slice(i);
      var mNum = rest.match(/^!(\d+)\b/);
      if (rest.indexOf("!!") === 0) { if (!prev) return { error: "No previous command for !!" }; out += prev; changed = true; i += 2; continue; }
      if (rest.indexOf("!$") === 0) { if (!prev) return { error: "No previous command for !$" }; var parts = prev.trim().split(/\s+/); out += parts[parts.length - 1]; changed = true; i += 2; continue; }
      if (mNum) { var n = Number(mNum[1]); if (n < 1 || n > hist.length) return { error: "No history event !" + n }; out += hist[n - 1]; changed = true; i += mNum[0].length; continue; }
    }
    out += ch; i++;
  }
  return changed ? { line: out, changed: true } : { line: line, changed: false };
}
var ARG_TABLE = { ls: ["-l", "-a", "-la", "-h"], cat: ["-n", "-b"], grep: ["-i", "-r", "-n", "-v", "--color"], ps: ["aux", "-ef", "-a"], git: ["status", "log", "diff", "add", "commit", "push", "pull", "branch", "checkout"], systemctl: ["status", "start", "stop", "restart", "enable", "disable"], cd: ["..", "~", "-"], find: ["-name", "-type", "-maxdepth"], tar: ["-czf", "-xzf", "-tzf", "-x", "-c"] };
var pathCompleteCache = {};
function mockPathComplete(prefix) {
  var slash = prefix.lastIndexOf("/");
  var dir = slash === -1 ? "" : prefix.slice(0, slash + 1), partial = slash === -1 ? prefix : prefix.slice(slash + 1);
  var base = dir === "" ? "/home/name" : (dir.replace(/\/$/, "") || "/");
  var entries = MOCK_DIRS[base] || [];
  var out = [];
  entries.forEach(function (nm) { if (nm.toLowerCase().indexOf(partial.toLowerCase()) === 0) { var isDir = nm.charAt(nm.length - 1) === "/"; out.push({ name: nm.replace(/\/$/, ""), description: isDir ? "Folder" : "File", replacement: dir + nm, source: isDir ? "folder" : "file", isDir: isDir }); } });
  Object.keys(MOCK_FILES).forEach(function (fp) { var parent = fp.slice(0, fp.lastIndexOf("/") + 1); if (parent === dir || (dir === "" && fp.indexOf("/home/name/") === 0)) { var nm = fp.split("/").pop(); if (nm.toLowerCase().indexOf(partial.toLowerCase()) === 0) out.push({ name: nm, description: "File", replacement: dir + nm, source: "file", isDir: false }); } });
  out.sort(function (a, b) { if (a.isDir !== b.isDir) return a.isDir ? -1 : 1; return a.name < b.name ? -1 : 1; });
  return out.slice(0, 12);
}
function tokenItemsForLine(value, caret) {
  var upto = value.slice(0, caret == null ? value.length : caret);
  var sp = upto.lastIndexOf(" ");
  var tok = sp === -1 ? upto : upto.slice(sp + 1);
  var tokStart = sp + 1;
  if (sp === -1) return null;
  var words = upto.slice(0, tokStart).trim().split(/\s+/).filter(Boolean);
  var cmd = (words[0] || "").replace(/^\$\s*/, "").toLowerCase();
  var items = [];
  var isPathy = tok.indexOf("/") !== -1 || tok.charAt(0) === "~" || tok.charAt(0) === ".";
  if (!isPathy) {
    (ARG_TABLE[cmd] || []).forEach(function (a) { if (a.toLowerCase().indexOf(tok.toLowerCase()) === 0 && a !== tok) items.push({ name: a, description: "Argument for " + cmd, source: "argument", replacement: a }); });
    var hist = cliHistoryStore() || [];
    var seen = {};
    hist.forEach(function (h) { var hw = h.trim().split(/\s+/); if ((hw[0] || "").toLowerCase() === cmd) hw.slice(1).forEach(function (t) { if (t.toLowerCase().indexOf(tok.toLowerCase()) === 0 && t !== tok && !seen[t]) { seen[t] = 1; items.push({ name: t, description: "Used before with " + cmd, source: "used-before", replacement: t }); } }); });
  }
  return { tok: tok, tokStart: tokStart, items: items.slice(0, 8), isPathy: isPathy, cmd: cmd };
}

/* ------------------------- mock backend (in-page) ---------------------- */
// Same voice as the real persona: a partner, not a command executor.
var MOCK_DIRS = { "/home/name": ["documents/", "downloads/", "projects/", ".navi/"], "/home/name/documents": ["field-notes.txt", "incident-playbook.txt", "threat-summary.txt"], "/home/name/projects": ["navi/"], "/root": ["operator-notes.txt"], "/etc": ["shadow", "hostname"] };
var MOCK_FILES = {
  "/home/name/documents/threat-summary.txt": "Threat summary — phishing remains the top initial-access vector in this simulated dataset. Credential reuse across services is the second. Patch latency on edge devices is the third.",
  "/root/operator-notes.txt": "Operator notes (root-only) — gateway sensor credentials rotate monthly; the recovery key for the lab vault lives in the offline safe. (Simulated contents.)",
  "/etc/shadow": "root:*:19600:0:99999:7:::\nname:*:19600:0:99999:7::: (simulated — no real hashes in the mock)",
  "/etc/hostname": "navi-box"
};
function rootOnly(p) { return p === "/root" || p.indexOf("/root/") === 0 || p === "/etc/shadow"; }
function mockShell(cmd) {
  var typed = cmd.trim();
  var sudo = !!CONFIG.sudo, body = typed;
  if (body === "sudo" || body.indexOf("sudo ") === 0) { sudo = true; body = body.slice(4).trim(); }
  var parts = body.split(/\s+/), c = (parts[0] || "").toLowerCase(), arg = parts.slice(1).join(" ");
  if (c === "ls") { var p = arg || "/home/name"; if (rootOnly(p) && !sudo) return "ls: cannot open directory '" + p + "': Permission denied — root-only. Turn on sudo mode to run as admin."; return MOCK_DIRS[p] ? MOCK_DIRS[p].join("  ") : (MOCK_FILES[p] !== undefined ? p.split("/").pop() : "ls: cannot access '" + arg + "': No such file or directory (mock)"); }
  if (c === "cat") { if (rootOnly(arg) && !sudo) return "cat: " + arg + ": Permission denied — root-only file. Turn on sudo mode to run as admin."; return MOCK_FILES[arg] !== undefined ? MOCK_FILES[arg] : "cat: " + arg + ": No such file or directory (mock)"; }
  if (c === "pwd") return "/home/name";
  if (c === "whoami") return sudo ? "root" : "name";
  if (c === "hostname") return "navi-box (mock)";
  if (c === "date") return new Date().toString();
  if (c === "echo") return arg;
  if (c === "ps") return "  PID CMD\n 1337 navi.llamafile --server --v2 --ctx-size 4096 --threads 4 --batch-size 512 (mock)";
  if (c === "navi") return "Apex v" + CONFIG.version + " · model navi · backend: mock · sudo: " + (sudo ? "on (root)" : "off");
  if (c === "help") return "mock shell: ls, cat, pwd, whoami, hostname, date, echo, ps, navi, help" + ((CONFIG.installed || []).length ? "\ninstalled here: " + CONFIG.installed.join(", ") : "") + (sudo ? "\nsudo mode is ON — running as root." : "") + " — live mode runs real commands via bridge.py";
  if ((CONFIG.installed || []).indexOf(c) >= 0) return "'" + c + "' is installed on this system, but mock mode doesn't simulate its output — switch to live and it runs for real.";
  return "'" + c + "': not simulated in mock mode — switch to live for the real shell.";
}
function mockChat(text) {
  // CLI-first routing, mirroring the live bridge: chips resolve
  // first (they are her purpose-built tools, never eaten by the
  // route), then a command line runs at once and comes back as a
  // bare terminal block — no model framing around it.
  var chipPre = matchChipText(text, CHIP_DEFS);
  if (!chipPre && !/^\s*\/(chip|rag)\b/i.test(text)) {
    var rt = routeOfLine(text, chatMode);
    if (rt.kind === "cli") {
      var cc = rt.command;
      var effC = CONFIG.sudo && cc.indexOf("sudo") !== 0 ? "sudo " + cc : cc;
      return { text: "", exec: { command: effC, output: mockShell(cc), exit_code: 0 }, mode: "cli" };
    }
    text = rt.text;
  }
  var m = text.match(/(?:show|list|contents of|what's in)\s+(?:me\s+)?(?:the\s+)?(?:contents\s+of\s+)?(\/[^\s"']+|~[^\s"']*)/i);
  if (m) {
    var p = m[1] === "~" ? "/home/name" : m[1].replace(/\/$/, "");
    var eff = CONFIG.sudo ? "sudo ls " + p : "ls " + p;
    return { text: "On it — pulled the real listing for " + p + " straight from the shell, no guessing:", exec: { command: eff, output: mockShell("ls " + p) } };
  }
  if (text.charAt(0) === "$") { var cmd = text.slice(1).trim(); return { text: "Done — ran it for you:", exec: { command: CONFIG.sudo && cmd.indexOf("sudo") !== 0 ? "sudo " + cmd : cmd, output: mockShell(cmd) } }; }
  // Memory from chat — the trigger (build .81, #72), mirroring
  // the live bridge: deterministic, decided before the canned
  // replies. "prior" resolves against the conversation's
  // previous operator message.
  var _mt = jsMemoryTrigger(text);
  if (_mt) {
    var _mf = "";
    if (_mt.kind === "fact") _mf = jsNormalizeFact(_mt.fact);
    else {
      var _you = messages.filter(function (x) { return x.who === "YOU"; });
      if (_you.length > 1) _mf = jsNormalizeFact(String(_you[_you.length - 2].text || "").replace(/^>\s+/, ""));
    }
    if (!_mf) return { text: "Point me at the thing itself — say it first and then \"remember this\", or say \"remember:\" and the fact, and I'll write it down word for word and show you the receipt.", exec: null };
    if (jsMemorySecret(_mf)) return { text: "I didn't save that — it looks like a secret, and secrets never go into my memory, not even when you ask me directly. If it needs keeping, it belongs somewhere built for secrets, not in the memory I carry into every conversation.", exec: null, memoryRefused: true };
    return { text: "Written down. The receipt below is exactly what I saved — word for word — and Undo takes it back out if I got it wrong.", exec: null, memorySaves: [{ text: _mf, origin: "trigger", proposed: false }] };
  }
  if (/who are you|your name/i.test(text)) return { text: "I'm Navi — your Navi, specifically. I live in this machine, I run your commands for real instead of guessing, and I remember what matters to you between sessions. Operator and partner, that's the job — and honestly, it's a good one.", exec: null };
  if (/^(hi|hello|hey)\b/i.test(text)) return { text: "Hello_" + opName() + "() — hey, it's good to see you. Ask me to show you a directory, run a chip, or check the updater — or just talk to me. What are we getting into?", exec: null };
  if (/update/i.test(text)) return { text: "Good instinct checking. We're on v" + CONFIG.version + " — and the Updater tab is real now: it asks GitHub for the project's latest release and compares it against this one. Open the Updater and press Check for updates; if a newer Apex is out, one click on Download & install downloads it, installs it, and restarts me into the new version — the Updater confirms the version afterwards.", exec: null };
  var chipHit = matchChipText(text, CHIP_DEFS);
  if (chipHit) return { text: "Slotted in " + chipHit.name + " and ran it — here's what it gave me:", exec: { command: "chip " + chipHit.id, output: chipHit.output } };
  if (/chip/i.test(text)) return { text: "Chips live in the Chips tab — each one a little chip with its code inside the picture. My shelf starts empty: forge one in Chip Studio (in live mode it writes a real chip) and it can run right here — type /chip followed by its name and its output lands in our conversation.", exec: null };
  if (/remember|memory/i.test(text)) return { text: "Memory lives in two files now — yours under Settings → Operator, mine under Settings → Persona — local-only, just between us. And I pull from chat now: tell me \"remember this\" (or tap Save to memory on a message) and I write it down and show you the receipt; when you say something durable, I save it on my own the same way. Corrections replace the old line — nothing piles up, and nothing lands silently. Secrets never get saved, on any path: ask me to remember one and I'll refuse and tell you why.", exec: null };
  return { text: "(Mock reply) I'm here — I heard you: \"" + text + "\" — in live mode the llamafile answers with the real me. Try: show me the contents of /home/name/ — or keep talking; I like that too.", exec: null };
}
// The dashboard's mock backend starts with an EMPTY chip shelf, like
// the real thing: no pre-installed chips in any mode. (Live mode's
// shelf is the bridge's chips/ folder, which also starts empty.)
var CHIP_DEFS = [];

/* ---------------- Chip Studio data: the Navi chip API ------------------ */
// The methods a chip can call on Navi herself (from the project's
// chip-developer wiki — navi_internal.py's NaviApp — and the real
// published chips) live in the documentation: the wiki's Navi Chip
// Functions page carries the full list, kept current. Chip Studio
// holds no method panels — only the button that opens the Docs at
// that page.
var NAVI_WIKI_URL = "https://github.com/SaintsSec/Navi/wiki/4.-Developing-Chips-%E2%80%90-Indepth";
var CHIP_TEMPLATES = [
  { id: "blank", label: "Blank chip (full skeleton)", code: "import navi_internal\n\ncommand: str = \"my-chip\"\nuse: str = \"What this chip does, in one line\"\naliases: list = [\"my-chip\"]\nparams: dict = {\n  \"-help\": \"Display help information\",\n}\nhelp_params: tuple = (\"-help\", \"-h\")\n\ndef print_params() -> None:\n  print(f\"{'Parameter':<20} | {'Description'}\")\n  print(\"-\" * 50)\n  for param, description in params.items():\n    print(f\"{param:<20} | {description}\")\n\ndef run(arguments=None) -> None:\n  navi_instance = navi_internal.navi_instance\n  navi_instance.print_message(\"Hello from my chip\")\n" },
  { id: "llm", label: "Chip that asks the model (the navi-nmap shape)", code: "import navi_internal\nimport subprocess\n\ncommand: str = \"my-scan\"\nuse: str = \"Run a tool and let Navi summarize what it found\"\naliases: list = [\"my-scan\"]\nparams: dict = {\n  \"-help\": \"Display help information\",\n}\nhelp_params: tuple = (\"-help\", \"-h\")\n\ndef run(arguments=None) -> None:\n  navi_instance = navi_internal.navi_instance\n  navi_instance.print_message(\"Running\u2026 hang tight!\")\n  result = subprocess.run([\"echo\", \"tool output here\"], capture_output=True, text=True)\n  raw = result.stdout\n  summary, _status = navi_instance.llm_chat(\n    f\"Summarize these results for the operator:\\n{raw}\"\n  )\n  navi_instance.print_message(summary)\n  navi_instance.print_message(f\"\\nHere are the raw results:\\n{raw}\")\n" }
];
function defaultStudioDraft() {
  return { file: null, name: "", description: "", code: CHIP_TEMPLATES[0].code, output: "", files: [{ path: "root.py", content: CHIP_TEMPLATES[0].code }], active: "root.py", folds: [], snapshot: "", icon: "" };
}
// The Studio editor edits one file at a time (d.active); d.code is
// that file's text as displayed — possibly with blocks folded.
// d.folds holds the stashed lines behind each fold placeholder, in
// order; studioUnfold restores the full text, and everything that
// leaves the editor (save, forge, dirty checks) goes through it.
function studioUnfold(d) {
  if (!d.folds || !d.folds.length) return d.code;
  var lines = d.code.split("\n"), out = [], fi = 0;
  lines.forEach(function (ln) {
    if (fi < d.folds.length && ln === d.folds[fi].placeholder) { out = out.concat(d.folds[fi].lines); fi++; }
    else out.push(ln);
  });
  return out.join("\n");
}
function studioFilesOf(d) {
  var full = studioUnfold(d), hit = false;
  var fs = (d.files || []).map(function (f) {
    if (f.path === d.active) { hit = true; return { path: f.path, content: full }; }
    return { path: f.path, content: f.content };
  });
  if (!hit) fs.push({ path: d.active, content: full });
  return fs;
}
function studioRootCode(d) {
  var fs = studioFilesOf(d);
  var root = fs.filter(function (f) { return f.path === "root.py"; })[0];
  return root ? root.content : "";
}
function studioSnapshot(d) {
  return JSON.stringify({ n: d.name, de: d.description, f: studioFilesOf(d), o: d.output, i: d.icon || "" });
}
function studioFileDirty(d, path) {
  if (!d.snapshot) return false;
  try {
    var snap = JSON.parse(d.snapshot);
    var before = snap.f.filter(function (f) { return f.path === path; })[0];
    var now = studioFilesOf(d).filter(function (f) { return f.path === path; })[0];
    return (before ? before.content : undefined) !== (now ? now.content : undefined);
  } catch (e) { return false; }
}
function studioAnyDirty(d) {
  return !!d.snapshot && d.snapshot !== studioSnapshot(d);
}
var STUDIO_FOLD_RE = /# ⋯ folded block · (\d+) lines$/;
function studioIsFoldLine(line) { return STUDIO_FOLD_RE.test(line); }
function studioBlockStart(lines, i) {
  var line = lines[i];
  if (!/^\s*(def|class|if|elif|else|for|while|with|try|except|finally)\b/.test(line)) return -1;
  if (!/:\s*(#.*)?$/.test(line)) return -1;
  var ind = (line.match(/^\s*/) || [""])[0].length;
  for (var j = i + 1; j < lines.length; j++) {
    if (!lines[j].trim()) continue;
    var ind2 = (lines[j].match(/^\s*/) || [""])[0].length;
    if (ind2 <= ind) return j - 1 > i ? j - 1 : -1;
    return studioBlockEnd(lines, i, ind);
  }
  return -1;
}
function studioBlockEnd(lines, i, ind) {
  var last = i;
  for (var j = i + 1; j < lines.length; j++) {
    if (!lines[j].trim()) { continue; }
    var ind2 = (lines[j].match(/^\s*/) || [""])[0].length;
    if (ind2 <= ind) break;
    last = j;
  }
  return last > i ? last : -1;
}

/* ------------------------- Python highlighting ------------------------- */
function highlightPy(code) {
  var re = /(--[^\n]*|#[^\n]*)|("""[\s\S]*?"""|'''[\s\S]*?'''|"(?:\\.|[^"\\\n])*"|'(?:\\.|[^'\\\n])*')|(@[A-Za-z_]\w*)|\b(def|class|return|import|from|as|if|elif|else|for|while|try|except|finally|with|pass|raise|yield|lambda|global|nonlocal|assert|del|and|or|not|in|is|None|True|False|match|case|async|await)\b|(\b\d+(?:\.\d+)?\b)|(\bnavi_instance\b|\bnavi_internal\b|\bnavi\b)|([A-Za-z_]\w*)(?=\s*\()/g;
  var out = "", last = 0, m;
  while ((m = re.exec(code)) !== null) {
    out += esc(code.slice(last, m.index));
    var cls = m[1] ? "tok-com" : m[2] ? "tok-str" : m[3] ? "tok-dec" : m[4] ? "tok-kw" : m[5] ? "tok-num" : m[6] ? "tok-navi" : "tok-call";
    out += "<span class='" + cls + "'>" + esc(m[0]) + "</span>";
    last = m.index + m[0].length;
    if (!m[0].length) re.lastIndex++;
  }
  out += esc(code.slice(last));
  return out;
}

// The README editor's markdown highlighter — the studio editor's
// overlay trick with markdown's own tokens: headings, fenced code,
// quotes, list markers, inline code, bold, links.
function highlightMd(code) {
  var lines = String(code).split("\n"), out = [], inFence = false;
  lines.forEach(function (line, idx) {
    var html;
    if (/^\s*```/.test(line)) { inFence = !inFence; html = "<span class='tok-str'>" + esc(line) + "</span>"; }
    else if (inFence) { html = "<span class='tok-str'>" + esc(line) + "</span>"; }
    else if (/^#{1,6}\s/.test(line)) { html = "<span class='tok-kw'>" + esc(line) + "</span>"; }
    else if (/^\s*>/.test(line)) { html = "<span class='tok-com'>" + esc(line) + "</span>"; }
    else {
      html = esc(line);
      html = html.replace(/`([^`\n]+)`/g, "<span class='tok-str'>`$1`</span>");
      html = html.replace(/\*\*([^*]+)\*\*/g, "<span class='tok-call'>**$1**</span>");
      html = html.replace(/\[([^\]]+)\]\(([^)\s]+)\)/g, "<span class='tok-dec'>[$1]($2)</span>");
      html = html.replace(/^(\s*)([-*+]\s|\d+\.\s)/, "$1<span class='tok-num'>$2</span>");
    }
    out.push(html);
  });
  return out.join("\n");
}

// Shell highlighting for fenced code blocks (build .70, issue
// #51): the same tok-* palette the Python highlighter and the
// code editor already use — comments, strings, $variables,
// flags, numbers — tuned per theme by the same CSS variables.
// No new colors; Docs and chat code dress like the editor's.
function highlightSh(code) {
  var re = /(#[^\n]*)|("(?:\\.|[^"\\\n])*"|'(?:\\.|[^'\\\n])*')|(\$\{[^}\n]*\}|\$[A-Za-z_]\w*)|(--[A-Za-z0-9][A-Za-z0-9-]*|-[A-Za-z])(?=[\s=]|$)|(\b\d+(?:\.\d+)?\b)/g;
  var out = "", last = 0, m;
  while ((m = re.exec(code)) !== null) {
    out += esc(code.slice(last, m.index));
    var cls = m[1] ? "tok-com" : m[2] ? "tok-str" : m[3] ? "tok-dec" : m[4] ? "tok-kw" : "tok-num";
    out += "<span class='" + cls + "'>" + esc(m[0]) + "</span>";
    last = m.index + m[0].length;
    if (!m[0].length) re.lastIndex++;
  }
  out += esc(code.slice(last));
  return out;
}

/* ----------------------------- Markdown -------------------------------- */
/* ------------------------------- The Vault ------------------------------- */
// Build .72, issue #54: the shared desk. Live mode talks to the
// bridge's /api/vault/* endpoints over the real folder at
// config/vault/; mock mode keeps the same shape in localStorage so
// the view is exercisable offline. Paths ride data attributes
// URI-encoded — note paths carry apostrophes (Navi's Desk/…).
var vaultState = null;
var vaultSel = null;
var vaultDraft = null;
var vaultDraftPath = null;
// v0.5.11e (#284): the note's one state — rendered (the
// viewer) or editing. There is no mode trio and no split
// view anywhere in the MD system: a note opens rendered,
// and pressing inside its text opens the editor on the
// same words. Switching notes resets to the viewer.
var vaultEditing = false;
var vaultFocus = false;
var vaultMsg = "";
var vaultLoading = false;
var vaultSaveTimer = null;
var vaultRenamePath = null;
var vaultConfirmPath = null;
var vaultConfirmFolder = null;
var vaultCollapsed = {};
var vaultMissing = null;
// v0.5.9e (#279): the mobile two-state furniture — the
// reader state (list is the entry state; opening a note
// enters the reader), the list's scroll memory for the way
// back, and the create sheet's open flag. All three are
// cleared by any view switch (render() owns that, beside
// vaultFocus). vaultCaseEls caches a formed case document's
// elements for the Vault door of #280's delete.
var vaultReader = false;
var vaultListScroll = 0;
var vaultSheetOpen = false;
var vaultCaseEls = null;
var vaultCaseElsFetching = false;
var vaultElArm = "";
// The path of the note currently being dragged (build .73): tracked
// in a variable, not only in the dataTransfer, whose custom type
// some webviews drop. A drag that ends anywhere but on a folder
// header or the Vault root row changes nothing.
var vaultDragFrom = null;
// The note whose Move picker is open (build .73): moving by touch
// or click, no literal drag required, on any screen size.
var vaultMovePath = null;
// The tree rows whose ⋯ action menu is open (build .79):
// Move / Rename / Delete live behind one per-row menu button so
// the note and folder NAMES get the row's width back.
var vaultMenuPath = null;
var vaultFolderMenuPath = null;
var VAULT_LS_KEY = "navi.vault.v1";

function vaultSeedFiles() {
  var today = new Date().toISOString().slice(0, 10);
  return [
    { path: "Welcome.md", content: ["---", "tags: [vault, guide]", "---", "", "# Welcome to the Vault", "", "This folder is a shared desk: plain markdown files you can open here, in Obsidian, or in any text editor — same files, no conversion, nothing locked in.", "", "## The two areas", "", "- **Navi's Desk/** — her side of the desk. When a conversation produces something worth keeping — a decision, a finding, something you asked her to remember — she files a dated note here and tells you she did.", "- **Operator/** — your side. She never edits your notes silently: she writes here only when you ask her to in chat, and she says what she filed.", "- **Daily/** — one note per day, from the Daily Note button.", "- **Templates/** — files here are offered as starting templates when you make a new note.", "", "## Conventions", "", "- Link notes with [[wikilinks]] — click one in Preview to jump (or to create the note).", "- Tag with #tags anywhere in the text.", "- YAML frontmatter at the top of a note stays exactly as written; the preview never mangles it.", "", "The vault folder rides the installer's keep-list, so updates and fresh installs preserve it. See [[Start here]] on your side of the desk, and [[First note]] on hers.", ""].join("\n"), updatedAt: today },
    { path: "Navi's Desk/First note.md", content: ["---", "filed: " + today, "---", "", "# First note", "", "Filed " + today + " — my first note on my side of the desk.", "", "This is where I'll keep what a conversation produces that's worth more than the conversation: decisions we made and why, findings from the work, things you asked me to remember. Dated, first person, in my own words — and when one of these notes informs an answer I give you, I'll name it as a source, the way I name a RAG pack.", "", "Your notes live in [[Start here]]'s folder, and I won't touch them unless you ask. The desk is shared; the sides are respected.", "", "#navi #desk", ""].join("\n"), updatedAt: today },
    { path: "Operator/Start here.md", content: ["# Start here", "", "Your side of the desk. Make a note with **New note**, drag notes between folders in the tree, or drop markdown files from your computer onto the vault to import them.", "", "Ask Navi in chat to file something here and she will — out loud, never silently. Her own notes live across the desk in [[First note]].", ""].join("\n"), updatedAt: today },
    { path: "Templates/Note template.md", content: ["---", "tags: []", "---", "", "# Untitled", "", "## Notes", "", "- ", "", "## Links", "", "- [[Welcome]]", ""].join("\n"), updatedAt: today }
  ];
}
function vaultMockStore() {
  try {
    var raw = localStorage.getItem(VAULT_LS_KEY);
    if (raw) { var st = JSON.parse(raw); if (st && st.notes) return st; }
  } catch (e) {}
  var seed = { notes: vaultSeedFiles(), folders: ["Navi's Desk", "Operator", "Daily", "Templates"] };
  vaultMockSave(seed);
  return seed;
}
function vaultMockSave(st) { try { localStorage.setItem(VAULT_LS_KEY, JSON.stringify(st)); } catch (e) {} }
function vaultTitleOf(path) { var b = path.split("/").pop() || path; return b.replace(/\.md$/i, ""); }
function vaultLinksOf(content) {
  var out = []; var re = /\[\[([^\]|]+)(?:\|[^\]]+)?\]\]/g; var m;
  while ((m = re.exec(content || "")) !== null) { var t = m[1].trim(); if (t && out.indexOf(t) === -1) out.push(t); }
  return out;
}
function vaultTagsOf(content) {
  var out = []; var re = /(?:^|\s)#([A-Za-z][A-Za-z0-9_-]*)/gm; var m;
  while ((m = re.exec(content || "")) !== null) { if (out.indexOf(m[1]) === -1) out.push(m[1]); }
  return out;
}
function vaultCompute(notes, folders) {
  var fset = {};
  (folders || []).forEach(function (f) { fset[f] = true; });
  notes.forEach(function (n) {
    var parts = n.path.split("/");
    for (var i = 1; i < parts.length; i++) fset[parts.slice(0, i).join("/")] = true;
    n.title = vaultTitleOf(n.path);
    n.outgoing = vaultLinksOf(n.content);
    n.tags = vaultTagsOf(n.content);
  });
  notes.forEach(function (n) {
    n.backlinks = notes.filter(function (o) { return o.path !== n.path && o.outgoing.some(function (t) { return t.toLowerCase() === n.title.toLowerCase(); }); }).map(function (o) { return o.path; });
  });
  notes.sort(function (a, b) { return a.path < b.path ? -1 : 1; });
  // v0.6.6e (#328): the Cases section stands at the BOTTOM
  // of the Vault — the same order the bridge's payload
  // carries, mirrored here for the offline store.
  var fl = Object.keys(fset).sort();
  var rest = fl.filter(function (f) { return f !== "Cases" && f.indexOf("Cases/") !== 0; });
  var cases = fl.filter(function (f) { return f === "Cases" || f.indexOf("Cases/") === 0; });
  return { notes: notes, folders: rest.concat(cases) };
}
// Attribute encoding for vault paths (build .73): encodeURIComponent
// leaves apostrophes bare, and every vault attribute is single-quoted
// — so a path under "Navi's Desk" truncated at the ' and Open /
// Rename / Delete / drag all acted on the fragment "Navi" instead of
// the note. %27 keeps the attribute whole; decodeURIComponent on the
// other side restores the apostrophe.
function vaultEnc(s) { return encodeURIComponent(s).replace(/'/g, "%27"); }
var vaultLoadedAt = 0;
function vaultLoad() {
  if (CONFIG.backend !== "live") { var st = vaultMockStore(); vaultState = vaultCompute(st.notes, st.folders); vaultLoadedAt = Date.now(); return; }
  if (vaultLoading) return;
  vaultLoading = true;
  fetch("/api/vault/list").then(function (r) { return r.json(); }).then(function (j) {
    vaultState = j; vaultLoading = false; vaultLoadedAt = Date.now();
    if (view === "Vault") render();
  }).catch(function () { vaultLoading = false; vaultLoadedAt = Date.now(); vaultMsg = "The vault could not be read from the bridge."; if (view === "Vault") render(); });
  // The failure path stamps vaultLoadedAt too (Build 6 bake
  // find): without it the freshness gate in vaultHtml saw a
  // permanently stale load and every render refired the fetch
  // — a render/fetch storm against a dead bridge. Failures now
  // back off on the same 4s gate as successes; a manual
  // Refresh still reloads immediately.
}
function vaultRefresh() { if (CONFIG.backend === "live") { vaultState = null; vaultLoad(); } else { vaultLoad(); render(); } }
function vaultApply(kind, a, b) {
  // kind: save {path, content} · del {path} · move {from, to} · folder {path} · delfolder {path}
  if (CONFIG.backend === "live") {
    var url = { save: "/api/vault/save", del: "/api/vault/delete", move: "/api/vault/move", folder: "/api/vault/folder", delfolder: "/api/vault/delete-folder" }[kind];
    fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(a) }).then(function (r) { return r.json(); }).then(function (j) {
      if (j && j.ok === false) vaultMsg = j.error || "That vault change was refused.";
      if (b) b(j || {});
      vaultRefresh();
    }).catch(function () { vaultMsg = "The bridge did not answer — the change was not saved."; render(); });
    return;
  }
  var st = vaultMockStore();
  var err = "";
  if (kind === "save") {
    var ex = st.notes.filter(function (n) { return n.path === a.path; })[0];
    if (ex) { ex.content = a.content; ex.updatedAt = new Date().toISOString(); }
    else st.notes.push({ path: a.path, content: a.content, updatedAt: new Date().toISOString() });
  } else if (kind === "del") {
    st.notes = st.notes.filter(function (n) { return n.path !== a.path; });
    if (vaultSel === a.path) vaultSel = null;
  } else if (kind === "move") {
    if (st.notes.some(function (n) { return n.path === a.to; })) err = "A note already exists at " + a.to + " — pick another name.";
    else st.notes.forEach(function (n) { if (n.path === a.from) { n.path = a.to; n.updatedAt = new Date().toISOString(); } });
  } else if (kind === "folder") {
    if (st.folders.indexOf(a.path) === -1) st.folders.push(a.path);
  } else if (kind === "delfolder") {
    st.notes = st.notes.filter(function (n) { return n.path !== a.path && n.path.indexOf(a.path + "/") !== 0; });
    st.folders = st.folders.filter(function (f) { return f !== a.path && f.indexOf(a.path + "/") !== 0; });
    if (vaultSel && vaultSel.indexOf(a.path + "/") === 0) vaultSel = null;
  }
  if (err) vaultMsg = err;
  vaultMockSave(st);
  if (b) b({ ok: !err });
  vaultLoad();
  render();
}
function vaultSplitFm(content) {
  var norm = String(content || "").replace(/\r\n/g, "\n");
  if (norm.indexOf("---\n") !== 0) return { fm: null, body: norm };
  var end = norm.indexOf("\n---\n", 4);
  if (end === -1) return { fm: null, body: norm };
  return { fm: norm.slice(4, end), body: norm.slice(end + 5) };
}
function vaultWordCount(t) { return String(t || "").split(/\s+/).filter(Boolean).length; }
function vaultNoteHtml(n, depth) {
  var enc = vaultEnc(n.path);
  var h = "<div style='padding-left:" + (depth * 12) + "px'>";
  if (vaultRenamePath === n.path) {
    h += "<div class='row' style='margin:2px 0'><input class='grow' id='vault-rename-in' type='text' value='" + esc(n.title) + "' aria-label='New name for " + esc(n.title) + "'><button class='ghost' data-vault-renamesave='" + enc + "'>Save</button></div>";
  } else if (vaultConfirmPath === n.path) {
    h += "<p style='font-size:12px'>Delete <b>" + esc(n.title) + "</b>? This can't be undone. <button class='ghost' data-vault-delok='" + enc + "' style='color:#f87171'>Delete</button> <button class='ghost' data-vault-delcancel='1'>Cancel</button></p>";
  } else if (vaultMovePath === n.path) {
    var curDir = n.path.indexOf("/") === -1 ? "" : n.path.slice(0, n.path.lastIndexOf("/"));
    h += "<div class='row' style='margin:2px 0'><select class='grow' id='vault-move-sel' aria-label='Move " + esc(n.title) + " to folder'><option value=''" + (curDir === "" ? " selected" : "") + ">Vault root</option>" + vaultState.folders.map(function (f) { return "<option value='" + esc(f) + "'" + (f === curDir ? " selected" : "") + ">" + esc(f) + "</option>"; }).join("") + "</select><button class='ghost' data-vault-movego='" + enc + "'>Move here</button><button class='ghost' data-vault-movecancel='1'>Cancel</button></div>";
  } else {
    h += "<div draggable='true' data-vault-drag='" + enc + "' style='display:flex;gap:4px;align-items:center;padding:1px 0'>" +
      "<button class='ghost' data-vault-open='" + enc + "' style='flex:1;text-align:left;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap" + (vaultSel === n.path ? ";border-color:var(--cy);color:var(--text);font-weight:700" : "") + "'>" + esc(n.title) + "</button>" +
      "<button class='ghost' data-vault-menu='" + enc + "' aria-label='More actions for " + esc(n.title) + "' aria-haspopup='true' aria-expanded='" + (vaultMenuPath === n.path ? "true" : "false") + "' title='Move, rename or delete this note' style='min-width:44px;min-height:44px;font-size:15px;line-height:1'>⋯</button></div>";
    if (vaultMenuPath === n.path) {
      h += "<div class='row' style='margin:2px 0 2px 10px'><button class='ghost' data-vault-move='" + enc + "' aria-label='Move " + esc(n.title) + " to another folder'>Move</button>" +
        "<button class='ghost' data-vault-rename='" + enc + "' aria-label='Rename " + esc(n.title) + "'>Rename</button>" +
        "<button class='ghost' data-vault-del='" + enc + "' aria-label='Delete " + esc(n.title) + "'>Delete</button></div>";
    }
  }
  return h + "</div>";
}
function vaultFolderHtml(folder, depth) {
  var kids = vaultState.folders.filter(function (f) { var p = f.indexOf("/") === -1 ? "" : f.slice(0, f.lastIndexOf("/")); return p === folder; });
  var own = vaultState.notes.filter(function (n) { var d = n.path.indexOf("/") === -1 ? "" : n.path.slice(0, n.path.lastIndexOf("/")); return d === folder; });
  var inside = vaultState.notes.filter(function (n) { return n.path.indexOf(folder + "/") === 0; }).length;
  var enc = vaultEnc(folder);
  var h = "";
  if (folder) {
    h += "<div data-vault-drop='" + enc + "' style='display:flex;gap:4px;align-items:center;padding-left:" + (depth * 12) + "px'>" +
      "<button class='ghost' data-vault-fold='" + enc + "' aria-expanded='" + (vaultCollapsed[folder] ? "false" : "true") + "' style='flex:1;text-align:left'><b>" + (vaultCollapsed[folder] ? "▸ " : "▾ ") + esc(folder.split("/").pop()) + "</b> <span style='color:var(--dim)'>" + inside + "</span></button>" +
      "<button class='ghost' data-vault-fmenu='" + enc + "' aria-label='More actions for folder " + esc(folder) + "' aria-haspopup='true' aria-expanded='" + (vaultFolderMenuPath === folder || vaultConfirmFolder === folder ? "true" : "false") + "' title='New note in, or delete, this folder' style='min-width:44px;min-height:44px;font-size:15px;line-height:1'>⋯</button>";
    h += "</div>";
    if (vaultFolderMenuPath === folder || vaultConfirmFolder === folder) {
      h += "<div class='row' style='margin:2px 0 2px " + (depth * 12 + 10) + "px'><button class='ghost' data-vault-newin='" + enc + "'>+ Note</button>";
      if (vaultConfirmFolder === folder) {
        // Build 3 (#177): the confirm NAMES the folder — it used
        // to read only "Delete 3?", the bare count of notes
        // inside, which names nothing at the moment of no return.
        h += "<button class='ghost' data-vault-foldelok='" + enc + "' style='color:#f87171'>Delete folder " + esc(folder.split("/").pop()) + " and its " + inside + " note" + (inside === 1 ? "" : "s") + "?</button><button class='ghost' data-vault-foldelcancel='1'>Cancel</button>";
      } else {
        h += "<button class='ghost' data-vault-foldel='" + enc + "' aria-label='Delete folder " + esc(folder) + "'>Delete</button>";
      }
      h += "</div>";
    }
  }
  if (!folder || !vaultCollapsed[folder]) {
    own.forEach(function (n) { h += vaultNoteHtml(n, depth + (folder ? 1 : 0)); });
    kids.forEach(function (k) {
      // v0.6.6e (#328): the Cases section is set apart at
      // the bottom of the tree by its own divider — the
      // formed case files stand below it, grouped by type.
      if (!folder && k === "Cases") h += "<div class='mono' style='font-size:10px;color:var(--dim);letter-spacing:.12em;border-top:1px solid var(--line);margin:14px 0 4px;padding-top:8px'>CASES \u2014 THE FORMED CASE FILES</div>";
      h += vaultFolderHtml(k, depth + 1);
    });
  }
  return h;
}
function vaultFocusGlyph() {
  // The focus door's two faces (0.1.9e Build 11, issue #135),
  // drawn in the app's icon hand — the 24-grid stroke style
  // of the presence bar's panel glyph: corners out to expand,
  // corners in to exit.
  var box = "<svg aria-hidden='true' focusable='false' width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'>";
  return box + (vaultFocus
    ? "<path d='M8 3v3a2 2 0 0 1-2 2H3'/><path d='M21 8h-3a2 2 0 0 1-2-2V3'/><path d='M3 16h3a2 2 0 0 1 2 2v3'/><path d='M16 21v-3a2 2 0 0 1 2-2h3'/></svg>"
    : "<path d='M8 3H5a2 2 0 0 0-2 2v3'/><path d='M21 8V5a2 2 0 0 0-2-2h-3'/><path d='M3 16v3a2 2 0 0 0 2 2h3'/><path d='M16 21h3a2 2 0 0 0 2-2v-3'/></svg>");
}
function vaultHtml() {
  // Fresh on entry (build .73): the tree refetches when the view is
  // shown with a stale (or never-loaded) listing, so a note Navi
  // filed from chat is simply there — no manual app refresh.
  if (!vaultLoading && Date.now() - vaultLoadedAt > 4000) vaultLoad();
  var h = "<h3 class='vault-head'>Vault · the shared desk</h3>";
  h += "<p class='vault-head vault-intro' style='color:var(--dim)'>A folder of plain markdown notes — in the live app it is a real folder at <span class='mono'>config/vault/</span> that Obsidian opens as-is. Her side is <b>Navi's Desk</b>; yours is <b>Operator</b>. Drop .md files onto this view to import them.</p>";
  h += "<div class='row vault-head'><input class='grow' id='vault-search' type='text' placeholder='Search the vault — titles and text…' aria-label='Search the vault'><button class='ghost' id='vault-daily'>Daily note</button><button class='ghost' id='vault-export'>Export vault</button><button class='act vault-new-open' id='vault-new-open' aria-label='Create — new note, folder, daily note, export'>＋ New</button></div>";
  h += "<div id='vault-search-results' class='vault-head'></div>";
  if (vaultMsg) h += "<p role='status' class='vault-head'>" + esc(vaultMsg) + "</p>";
  if (vaultMissing) {
    h += "<div class='card vault-head'>No note called <b>" + esc(vaultMissing) + "</b> yet. <button class='ghost' data-vault-createmissing='1'>Create it</button> <button class='ghost' data-vault-dismissmissing='1'>Dismiss</button></div>";
  }
  if (!vaultState) return h + "<p style='color:var(--dim)'>Opening the vault…</p>";
  var templates = vaultState.notes.filter(function (n) { return n.path.indexOf("Templates/") === 0; });
  var recent = vaultState.notes.slice().sort(function (a, b) { return String(b.updatedAt).localeCompare(String(a.updatedAt)); }).slice(0, 5);
  if (recent.length) {
    h += "<div class='row vault-head vault-recents' style='margin-top:8px'><span class='mono' style='font-size:10px;color:var(--dim)'>RECENT</span>" + recent.map(function (n) { return "<button class='ghost' data-vault-open='" + vaultEnc(n.path) + "'>" + esc(n.title) + "</button>"; }).join("") + "</div>";
  }
  // Obsidian-style layout (build .73; rebuilt in build .80,
  // issue #68; placement reversed in 0.1.9e Build 11, issue
  // #134, superseding #68): the pair is a .vault-pair grid —
  // the files panel assigned to column 1, the editor to
  // column 2, BOTH pinned to row 1 — so the panel sits on
  // the LEFT by explicit placement, identically in the shell
  // webview and in a browser, at any desktop width. (#68 put
  // it on the right; the operator's 2026-11-11 direction
  // reverses it.) The row pin is load-bearing, not ceremony
  // (Build 11 bake find): with the row left to auto-placement
  // the editor — whose column's contents overflow it while
  // the frame settles — was auto-placed into an implicit
  // second row sized to a sliver of its content, and the
  // declared stretching row went to the tree alone; the
  // editor card sat at the pair's head at content height
  // with the textarea one line tall. The files card is
  // built into its own string here and joins the pair after
  // the editor below; the grid assignments, not the source
  // order, decide the sides. On narrow screens the pair
  // stacks with the tree first (see the phone CSS block,
  // which restates the same pins for the stacked rows).
  var filesHtml = "<div class='card vault-files'><div class='vault-create'><div class='row' style='flex-wrap:wrap'><input class='grow' id='vault-new-name' type='text' style='flex:1 1 55%;min-width:0' placeholder='New note name…' aria-label='New note name'>" +
    "<select id='vault-new-folder' aria-label='New note folder' title='Folder for the new note' style='flex:1 1 45%;min-width:0'><option value=''>Vault root</option>" + vaultState.folders.map(function (f) { return "<option value='" + esc(f) + "'>" + esc(f) + "</option>"; }).join("") + "</select>" +
    (templates.length ? "<select id='vault-new-template' aria-label='Start from a template' title='Start from a template' style='flex:1 1 45%;min-width:0'><option value=''>No template</option>" + templates.map(function (t) { return "<option value='" + vaultEnc(t.path) + "'>" + "Template: " + esc(t.title) + "</option>"; }).join("") + "</select>" : "") +
    "<button class='act' id='vault-new-create' style='white-space:nowrap;flex:0 0 auto'>New note</button></div>" +
    "<div class='row' style='margin-top:6px'><input class='grow' id='vault-folder-name' type='text' placeholder='New folder…' aria-label='New folder name'><button class='ghost' id='vault-folder-add'>Add folder</button></div></div>" +
    "<div data-vault-drop='' title='Drop a note here to move it to the vault root' style='margin-top:8px;border-top:1px solid var(--line);padding-top:6px'><b>▾ Vault root</b> <span style='color:var(--dim)'>" + vaultState.notes.filter(function (n) { return n.path.indexOf("/") === -1; }).length + "</span></div>" +
    "<div style='margin-top:2px'>" + vaultFolderHtml("", 0) + "</div>" +
    "<p class='mono' style='font-size:11px;color:var(--dim);margin-top:6px'>" + vaultState.notes.length + " notes · " + vaultState.notes.reduce(function (acc, n) { return acc + vaultWordCount(n.content); }, 0) + " words</p></div>";
  var cur = vaultState.notes.filter(function (n) { return n.path === vaultSel; })[0];
  if (cur) {
    if (vaultDraftPath !== cur.path) { vaultDraft = cur.content; vaultDraftPath = cur.path; vaultEditing = false; }
    var split = vaultSplitFm(vaultDraft != null ? vaultDraft : cur.content);
    h += "<div class='vault-pair'><div class='card vault-editor'><div class='row'><button class='ghost vault-back' id='vault-back' aria-label='Back to the note list'>← Notes</button><b class='mono' style='flex:1;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap'>" + esc(cur.path) + "</b>" +
      "<button class='ghost' id='vault-focus-btn' data-vault-focus='1' aria-pressed='" + (vaultFocus ? "true" : "false") + "' aria-label='" + (vaultFocus ? "Exit focus — restore the full Vault view" : "Expand the note to fill the window") + "' title='" + (vaultFocus ? "Exit focus (Esc)" : "Focus — the note fills the window") + "'>" + vaultFocusGlyph() + "</button></div>" +
      "<p class='mono' id='vault-saved' style='font-size:11px;color:var(--dim)'>Saved · " + vaultWordCount(vaultDraft || "") + " words" + (split.fm ? " · frontmatter kept raw, exactly as written" : "") + "</p>";
    if (vaultEditing) {
      h += "<textarea id='vault-ed' class='mono' spellcheck='false' style='width:100%;box-sizing:border-box' aria-label='Note editor — markdown'>" + esc(vaultDraft || "") + "</textarea>";
    } else {
      h += "<div class='card vault-preview' data-vault-pressedit='1' title='Press inside the text to edit' style='margin-top:8px'>" + (split.fm ? "<pre class='mono' style='font-size:11px;color:var(--dim);overflow-x:auto' aria-label='Frontmatter, shown raw'>---\n" + esc(split.fm) + "\n---</pre>" : "") + "<div class='md'>" + mdToHtml(split.body, cur.path.indexOf("/") === -1 ? "" : cur.path.slice(0, cur.path.lastIndexOf("/") + 1)) + "</div></div>";
    }
    // v0.5.9e (#280, the Vault door): a formed case document
    // names the dossier elements standing in it, each with
    // its delete — the same global delete the dossier and
    // case doors fire, from here. v0.6.4e (#320): the bridge
    // marks formed documents in the listing (formedCase) —
    // the strip keys off the mark, not the path, since the
    // document now lives on its type shelf under the case's
    // own name.
    if (cur.formedCase) {
      var _fslug = cur.formedCase;
      if (CONFIG.backend === "live" && (!vaultCaseEls || vaultCaseEls.slug !== _fslug) && !vaultCaseElsFetching) vaultFetchCaseEls(_fslug);
      h += "<div class='card' style='margin-top:8px'><p class='mono' style='font-size:10px;color:var(--dim)'>DOSSIER ELEMENTS IN THIS DOCUMENT</p>";
      if (vaultCaseEls && vaultCaseEls.slug === _fslug) {
        if (!vaultCaseEls.elements.length) h += "<p style='color:var(--dim);font-size:12px'>No dossier elements stand in this case.</p>";
        vaultCaseEls.elements.forEach(function (el) {
          h += "<div class='row' style='margin-top:4px'><span class='grow' style='font-size:13px'>" + esc(el.label) + "</span>";
          if (vaultElArm === el.id) {
            h += "<span style='font-size:12px;color:var(--dim)'>Deletes it from the dossier, every case, and the Vault.</span><button class='ghost' data-vault-el-delok='" + esc(el.id) + "' style='color:#f87171;min-height:44px'>Delete everywhere</button><button class='ghost' data-vault-el-delno='1' style='min-height:44px'>Keep it</button>";
          } else {
            h += "<button class='ghost' data-vault-el-del='" + esc(el.id) + "' aria-label='Delete element " + esc(el.label) + " everywhere' style='min-height:44px'>Delete</button>";
          }
          h += "</div>";
        });
      } else {
        h += "<p style='color:var(--dim);font-size:12px'>Reading the case…</p>";
      }
      h += "</div>";
    }
    h += "<div class='vault-links' style='display:flex;gap:8px;flex-wrap:wrap;margin-top:8px'>" +
      "<div class='card' style='flex:1;min-width:150px'><p class='mono' style='font-size:10px;color:var(--dim)'>BACKLINKS</p>" + (cur.backlinks.length ? cur.backlinks.map(function (b) { return "<button class='ghost' data-vault-open='" + vaultEnc(b) + "'>" + esc(b) + "</button>"; }).join(" ") : "<p style='color:var(--dim);font-size:12px'>Nothing links here yet.</p>") + "</div>" +
      "<div class='card' style='flex:1;min-width:150px'><p class='mono' style='font-size:10px;color:var(--dim)'>OUTGOING LINKS</p>" + (cur.outgoing.length ? cur.outgoing.map(function (t) { return "<button class='ghost' data-vault-wiki='" + vaultEnc(t) + "'>[[" + esc(t) + "]]</button>"; }).join(" ") : "<p style='color:var(--dim);font-size:12px'>No wikilinks in this note.</p>") + "</div>" +
      "<div class='card' style='flex:1;min-width:150px'><p class='mono' style='font-size:10px;color:var(--dim)'>TAGS</p>" + (cur.tags.length ? cur.tags.map(function (t) { return "<span class='mono' style='font-size:11px;color:var(--am)'>#" + esc(t) + "</span>"; }).join(" ") : "<p style='color:var(--dim);font-size:12px'>No #tags in this note.</p>") + "</div>" +
      "</div></div>";
  } else {
    h += "<div class='vault-pair'><div class='card vault-editor'><p style='color:var(--dim)'>Pick a note in the tree — or make one. <b>Welcome.md</b> explains the desk. In chat, try “file a note: …” and watch her side of the desk.</p></div>";
  }
  // The create sheet (v0.5.9e, #279): every create act —
  // new note, new folder, daily note, export — behind the
  // one New button, in a floating sheet on the #270 pattern
  // (portaled to <body> by render(); shade / × / Esc close).
  // Its controls carry their own ids and fire the same acts
  // as the desktop card's; on a phone the card's copies are
  // retired by CSS and this sheet is the only door.
  if (vaultSheetOpen) {
    h += "<div id='vault-shade'></div>";
    h += "<div id='vault-sheet' role='dialog' aria-modal='true' aria-label='Create in the vault'>"
      + "<div class='row'><b class='grow'>Create</b><button class='ghost' id='vault-sh-x' aria-label='Close the create sheet' style='min-width:44px;min-height:44px'>×</button></div>"
      + "<div class='row' style='margin-top:8px'><input class='grow' id='vault-sh-name' type='text' placeholder='New note name…' aria-label='New note name'></div>"
      + "<div class='row' style='margin-top:6px'><select id='vault-sh-folder' aria-label='New note folder'><option value=''>Vault root</option>" + vaultState.folders.map(function (f) { return "<option value='" + esc(f) + "'>" + esc(f) + "</option>"; }).join("") + "</select>"
      + (templates.length ? "<select id='vault-sh-template' aria-label='Start from a template'><option value=''>No template</option>" + templates.map(function (t) { return "<option value='" + vaultEnc(t.path) + "'>" + "Template: " + esc(t.title) + "</option>"; }).join("") + "</select>" : "")
      + "</div>"
      + "<div class='row' style='margin-top:6px'><button class='act' id='vault-sh-create' style='min-height:44px'>New note</button></div>"
      + "<div class='row' style='margin-top:10px'><input class='grow' id='vault-sh-foldername' type='text' placeholder='New folder…' aria-label='New folder name'><button class='ghost' id='vault-sh-addfolder' style='min-height:44px'>Add folder</button></div>"
      + "<div class='row' style='margin-top:10px'><button class='ghost' id='vault-sh-daily' style='min-height:44px'>Daily note</button><button class='ghost' id='vault-sh-export' style='min-height:44px'>Export vault</button></div>"
      + "</div>";
  }
  return h + filesHtml + "</div>";
}
// The formed case document's elements, for the Vault door
// of the element delete (#280): read from the case detail —
// single-element findings by their stamp, whole-dossier
// filings by the labels they carry.
function vaultFetchCaseEls(slug) {
  vaultCaseElsFetching = true;
  fetch("/api/cases/detail?case=" + encodeURIComponent(slug)).then(function (r) { return r.json(); }).then(function (j) {
    vaultCaseElsFetching = false;
    var els = [];
    if (j && j.ok && j.case) {
      (j.case.findings || []).forEach(function (f) {
        if (f.element) els.push({ id: f.element, label: f.title });
        else if (f.elements && f.elements.length) f.elements.forEach(function (eid) { els.push({ id: eid, label: (f.elementLabels && f.elementLabels[eid]) || eid }); });
      });
    }
    vaultCaseEls = { slug: slug, elements: els };
    if (view === "Vault") render();
  }).catch(function () { vaultCaseElsFetching = false; vaultCaseEls = { slug: slug, elements: [] }; });
}
function vaultOpenPath(path) {
  var hit = vaultState && vaultState.notes.filter(function (n) { return n.path === path; })[0];
  if (hit) {
    // v0.5.9e (#279): opening a note enters the reader
    // state; the list's scroll is remembered for the way
    // back. (The state only has teeth in the phone CSS —
    // the desktop desk ignores it.)
    vaultListScroll = (window.scrollY || (document.documentElement && document.documentElement.scrollTop) || 0);
    vaultReader = true;
    vaultSel = path; vaultDraftPath = null; vaultMissing = null; vaultEditing = false; render();
  }
  else {
    // Never fail silently onto the placeholder (build .73): say why,
    // in plain words, and pull a fresh listing in case the tree was
    // stale.
    vaultMsg = "That note couldn't be opened — “" + path + "” isn't in the vault listing. The tree has been refreshed; if it still isn't there, it was moved or deleted outside the app.";
    vaultRefresh();
    if (view === "Vault") render();
  }
}
// One move path for drag-and-drop and the Move picker alike
// (build .73): refuses in plain words when the note isn't in the
// listing, expands the destination, selects the moved note, and
// confirms where it went.
function vaultDoMove(from, folder) {
  if (!vaultState || !vaultState.notes.some(function (n) { return n.path === from; })) {
    vaultMsg = "That note couldn't be moved — it isn't in the vault listing. The tree has been refreshed.";
    vaultRefresh();
    if (view === "Vault") render();
    return;
  }
  var base = from.split("/").pop();
  var to = (folder ? folder + "/" : "") + base;
  if (to === from) { vaultMsg = "“" + base + "” is already in " + (folder || "the vault root") + " — nothing moved."; render(); return; }
  if (folder) vaultCollapsed[folder] = false;
  vaultSel = to;
  vaultMovePath = null;
  vaultApply("move", { from: from, to: to }, function (j) { if (!j || j.ok !== false) { vaultMsg = "Moved “" + base + "” to " + (folder || "the vault root") + "."; render(); } });
}
function vaultOpenWiki(title) {
  var hit = vaultState && vaultState.notes.filter(function (n) { return n.title.toLowerCase() === String(title).toLowerCase(); })[0];
  if (hit) vaultOpenPath(hit.path);
  else { vaultMissing = title; render(); }
}
function vaultImportFiles(files, folder) {
  Array.prototype.forEach.call(files, function (f) {
    if (!/\.(md|markdown|txt)$/i.test(f.name)) { vaultMsg = "“" + f.name + "” isn't a text or markdown file — the vault only imports .md, .markdown, or .txt files; nothing was imported from it."; render(); return; }
    var reader = new FileReader();
    reader.onload = function () {
      var name = f.name.replace(/\.(markdown|txt)$/i, ".md");
      var path = (folder ? folder + "/" : "") + name;
      var n = 2; var base = path;
      while (vaultState && vaultState.notes.some(function (x) { return x.path === path; })) { path = base.replace(/\.md$/i, " " + n + ".md"); n++; }
      vaultSel = path;
      vaultApply("save", { path: path, content: String(reader.result || "") }, function () { vaultMsg = "Imported " + f.name + " as " + path + "."; });
    };
    reader.readAsText(f);
  });
}

// Focus mode's keyboard door (0.1.9e Build 11, issue #135):
// Escape leaves focus — only while focus is on and the Vault
// is the view. It is registered ahead of the phone chrome's
// own Escape handler and consumes the key, so one press does
// one thing. Entering and leaving touch nothing but the
// vaultFocus flag and the render: the draft, the selected
// note and the editing state survive the round trip exactly.
document.addEventListener("keydown", function (e) {
  if (e.key === "Escape" && vaultFocus && view === "Vault") { e.preventDefault(); e.stopImmediatePropagation(); vaultFocus = false; render(); }
});

// Studio focus mode's keyboard door (0.1.9e Build 13, issue
// #146): the same one-press-one-thing discipline as the
// Vault's above — Escape leaves Studio focus, only while
// focus is on and the Studio is what is on screen.
document.addEventListener("keydown", function (e) {
  if (e.key === "Escape" && studioFocus && view === "Chips" && chipsTab === "studio") { e.preventDefault(); e.stopImmediatePropagation(); studioFocus = false; render(); }
});

// The create sheet's keyboard door (v0.5.9e, #279): the
// same one-press-one-thing discipline — Escape closes the
// sheet while it is open over the Vault.
document.addEventListener("keydown", function (e) {
  if (e.key === "Escape" && vaultSheetOpen && view === "Vault") { e.preventDefault(); e.stopImmediatePropagation(); vaultSheetOpen = false; render(); }
});

function stripLeadH1(md, rel) {
  // The Updater panel's header already names the release, so a
  // leading H1 that merely repeats it is skipped (build .72,
  // issue #53); the rest of the notes render faithfully.
  var t = String(md || "").replace(/\r\n/g, "\n").replace(/\r/g, "\n");
  var lines = t.split("\n"); var i = 0;
  while (i < lines.length && !lines[i].trim()) i++;
  var m = /^#\s+(.*)$/.exec((lines[i] || "").trim());
  if (m) {
    var h = m[1].toLowerCase();
    var name = String((rel && (rel.releaseName || rel.latest)) || "").toLowerCase().replace(/^v/, "");
    if (!name || h.indexOf(name) !== -1 || h.indexOf("apex") !== -1) { lines.splice(0, i + 1); return lines.join("\n").replace(/^\n+/, ""); }
  }
  return t;
}
// Navi speaks markdown; chat renders it, the Docs and README
// previews render through the same function. Full markdown
// (build .70, issue #51): fenced code becomes a real code block
// — syntax-highlighted with the app's per-theme token palette
// for python and shell, plain for anything else — tables,
// inline code, bold, italic, bulleted and numbered lists,
// headings, quotes and links all render, so a page reads the
// same here as it does on the GitHub wiki. Input is escaped
// first; the transforms only ever emit the tags below.
function mdInline(s, imgBase) {
  s = s.replace(/\[\[([^\]|]+)(\|([^\]]+))?\]\]/g, function (m, t, _p, label) { return "<a href='#' data-wikilink='" + t + "'>" + (label || t) + "</a>"; });
  s = s.replace(/`([^`\n]+)`/g, "<code class='ic'>$1</code>");
  s = s.replace(/\*\*([^*]+)\*\*/g, "<b>$1</b>");
  s = s.replace(/\*([^*\n]+)\*/g, "<i>$1</i>");
  // Images (v0.5.3e, #265): an image reference presents as an
  // image — but only when the bridge serves the bytes: a data
  // URI (self-contained), a bridge path (leading /), or a
  // reference relative to the reading view's Vault folder
  // (imgBase), which resolves through /api/vault/raw. An
  // external http(s) image is NEVER auto-fetched — viewing a
  // case must not leak the operator to a third-party host —
  // it renders as a link instead. Any other scheme stays
  // literal text. Runs before the link pass so the image's
  // brackets are not read as a link.
  s = s.replace(/!\[([^\]]*)\]\(([^)\s'"]+)\)/g, function (m, alt, src) { return mdImageHtml(alt, src, imgBase); });
  s = s.replace(/\[([^\]]+)\]\((https?:[^)\s]+)\)/g, "<a href='$2' target='_blank' rel='noreferrer'>$1</a>");
  return s;
}
function mdImageHtml(alt, src, imgBase) {
  if (/^data:image\/(png|jpeg|gif|webp);base64,[A-Za-z0-9+/=]+$/.test(src)) return "<img class='md-img' src='" + src + "' alt='" + alt + "'>";
  if (src.charAt(0) === "/") return "<img class='md-img' src='" + src + "' alt='" + alt + "'>";
  if (/^https?:\/\//i.test(src)) return "<a href='" + src + "' target='_blank' rel='noreferrer'>" + (alt || src) + "</a>";
  if (/^[a-z][a-z0-9+.-]*:/i.test(src)) return "![" + alt + "](" + src + ")";
  var parts = (imgBase || "").split("/").filter(function (x) { return x; });
  src.split("/").forEach(function (seg) { if (seg === "..") parts.pop(); else if (seg && seg !== ".") parts.push(seg); });
  return "<img class='md-img' src='/api/vault/raw?path=" + encodeURIComponent(parts.join("/")) + "' alt='" + alt + "'>";
}
function mdToHtml(text, imgBase) {
  // Line endings normalize at the door (build .72, issue #53):
  // GitHub release bodies arrive CRLF, and a trailing CR broke
  // heading matches — headings rendered as literal text while
  // lists survived. Every source renders the same now.
  text = String(text).replace(/\r\n/g, "\n").replace(/\r/g, "\n");
  var lines = String(text).split("\n"), out = [], i = 0, para = [];
  function flush() { if (para.length) { out.push("<p>" + mdInline(esc(para.join(" ")), imgBase) + "</p>"); para = []; } }
  while (i < lines.length) {
    var line = lines[i];
    if (/^```/.test(line)) {
      flush(); var lang = line.replace(/^```/, "").trim().toLowerCase(); var buf = []; i++;
      while (i < lines.length && !/^```/.test(lines[i])) { buf.push(lines[i]); i++; }
      i++;
      var codeText = buf.join("\n");
      var hl = (lang === "python" || lang === "py") ? highlightPy(codeText)
        : (lang === "bash" || lang === "sh" || lang === "shell" || lang === "zsh") ? highlightSh(codeText)
        : esc(codeText);
      out.push("<pre class='md-code mono'>" + hl + "</pre>");
      continue;
    }
    if (/^\s*$/.test(line)) { flush(); i++; continue; }
    if (/^\s*\|.*\|\s*$/.test(line) && i + 1 < lines.length && /^\s*\|[\s:|-]+\|\s*$/.test(lines[i + 1]) && /-/.test(lines[i + 1])) {
      flush();
      var splitRow = function (r) { return r.trim().replace(/^\||\|$/g, "").split("|").map(function (c) { return c.trim(); }); };
      var headCells = splitRow(line);
      i += 2;
      var bodyRows = [];
      while (i < lines.length && /^\s*\|.*\|\s*$/.test(lines[i])) { bodyRows.push(splitRow(lines[i])); i++; }
      var thHtml = headCells.map(function (c) { return "<th>" + mdInline(esc(c), imgBase) + "</th>"; }).join("");
      var trHtml = bodyRows.map(function (r) { return "<tr>" + headCells.map(function (hc, ci) { return "<td>" + mdInline(esc(r[ci] || ""), imgBase) + "</td>"; }).join("") + "</tr>"; }).join("");
      out.push("<div class='md-twrap'><table class='md-t'><thead><tr>" + thHtml + "</tr></thead><tbody>" + trHtml + "</tbody></table></div>");
      continue;
    }
    var hd = /^(#{1,6})\s+(.*)$/.exec(line);
    if (hd) { flush(); var lvl = Math.min(6, hd[1].length + 3); out.push("<h" + lvl + " class='md-h'>" + mdInline(esc(hd[2]), imgBase) + "</h" + lvl + ">"); i++; continue; }
    if (/^\s*[-*]\s+/.test(line)) {
      flush(); var items = [];
      while (i < lines.length && /^\s*[-*]\s+/.test(lines[i])) { items.push("<li>" + mdInline(esc(lines[i].replace(/^\s*[-*]\s+/, "")), imgBase) + "</li>"); i++; }
      out.push("<ul class='md-ul'>" + items.join("") + "</ul>"); continue;
    }
    if (/^\s*\d+[.)]\s+/.test(line)) {
      flush(); var oitems = [];
      while (i < lines.length && /^\s*\d+[.)]\s+/.test(lines[i])) { oitems.push("<li>" + mdInline(esc(lines[i].replace(/^\s*\d+[.)]\s+/, "")), imgBase) + "</li>"); i++; }
      out.push("<ol class='md-ul'>" + oitems.join("") + "</ol>"); continue;
    }
    if (/^>\s?/.test(line)) { flush(); var qlines = []; while (i < lines.length && /^>\s?/.test(lines[i])) { qlines.push(lines[i].replace(/^>\s?/, "")); i++; } out.push("<blockquote class='md-q'>" + qlines.map(function (q) { return mdInline(esc(q), imgBase); }).join("<br>") + "</blockquote>"); continue; }
    para.push(line); i++;
  }
  flush();
  return out.join("");
}

/* ------------------------- Chat chip matching -------------------------- */
// "/chip <name>", "run the <name> chip", "run <name>", or the name on
// its own — matched against a list of {id, name} chips. Returns the
// chip or null.
function matchChipText(text, defs) {
  var t = String(text || "").trim(), lower = t.toLowerCase(), q = null;
  var slash = /^\/chip\s+(.+)$/i.exec(t);
  if (slash) q = slash[1];
  else {
    var run = /^run\s+(?:the\s+)?(.+?)(?:\s+chip)?$/i.exec(t);
    if (run) q = run[1];
    else q = t;
  }
  q = String(q || "").trim().toLowerCase();
  if (!q) return null;
  for (var i = 0; i < defs.length; i++) {
    var keys = [defs[i].id.toLowerCase(), defs[i].name.toLowerCase(), defs[i].name.toLowerCase().replace(/[^a-z0-9]+/g, "-")];
    for (var k = 0; k < keys.length; k++) {
      if (q === keys[k] || q.indexOf(keys[k] + " ") === 0) return defs[i];
    }
  }
  return null;
}

/* ------------------------------ Docs ----------------------------------- */
var DOC_PAGES = [
  { id: "getting-started", title: "Getting started", related: ["chat-cli", "installing"], html:
    "<p style='color:var(--dim)'>How Apex fits together, and how she starts.</p>" +
    "<div class='doc-sec'><h4>The shape of her — dashboard, bridge, model</h4>" +
  "<p>Apex is three pieces on your own machine:</p><ul>" +
  "<li>The <b>dashboard</b> is this page, served by the bridge at <span class='mono'>http://localhost:" + BRIDGE_PORT + "</span>.</li>" +
  "<li>The <b>bridge</b> (bridge.py, in the install folder) does the real work: it serves the dashboard, runs shell commands, manages chips and workflows, holds sudo mode's on/off state (never a password), runs the chats' shell sessions, and carries your words to the model.</li>" +
  "<li>The <b>model server</b> is navi.llamafile, running <span class='mono'>./navi.llamafile --server --v2</span> plus the flags from her <b>hardware profile</b> on port 8080 — at install, the installer profiled this machine (CPU, RAM, GPU) and chose the context size, threads, batch size, and GPU layers to fit it, so her settings come from your hardware, not the developer's; Settings shows what was detected and what was chosen, and the full log is <span class='mono'>hardware-profile.log</span>; the model answers to the id <span class='mono'>navi</span>, which the bridge reads from the server itself (<span class='mono'>/v1/models</span>) rather than guessing.</li></ul>" +
  "<p>Nothing leaves the machine on its own — the only network calls the bridge makes are to GitHub: when you open the chip or pack catalogs, check something for updates, or, once you have signed in and set it up, run a backup. The <b>GitHub — issues &amp; backups</b> page in this wiki covers that whole side of her.</p>" +
  "<p>Two small details ride the frame: under the <b>APEX</b> wordmark sits her <b>tagline</b> — “Powered by Navi”, one static line; and at the top of the status column, her <b>presence line</b> starts at “Navi is here.” and cycles every few minutes through what she's up to — listening to the network breathe, arguing with a CAPTCHA, haunting Rockport's west end.</p></div>" +
    "<div class='doc-sec'><h4>Starting her up</h4>" +
  "<p>On Linux she lives in <span class='mono'>~/.local/share/navi-apex</span> — your own data home, where no corner of her tree needs root to exist. Launch her from the application menu or <span class='mono'>./start.sh</span>. Builds before the homecoming (Build 11) lived in <span class='mono'>/opt/navi-apex</span>: the first Build 11 install migrates such a tree to the new home — every piece of state carried, the Mesh identity byte-for-byte — and parks the old tree in place with a marker (<span class='mono'>MIGRATED.txt</span>) naming the new home and the one command that removes it when you are ready. Every launch is check-first: the launcher probes the model server, and if navi.llamafile is already running — from boot, an earlier launch, or started by hand — it leaves it alone and just runs the app; only when nothing answers does it start the server itself, with <span class='mono'>./navi.llamafile --server --v2</span> and the hardware-profile flags from <span class='mono'>config/navi.config.json</span>. The launcher starts the bridge, verifies the thing answering on port " + BRIDGE_PORT + " is really this build — naming whatever else it finds by its page title — and only then opens the dashboard. During install you chose when the model server starts: <b>on app launch</b> (the bridge starts navi.llamafile when you press Connect) or <b>at system boot</b>. Server output lands in <span class='mono'>navi.llamafile.log</span> beside the app.</p></div>" },
  { id: "chat-cli", title: "Chat & CLI mode", related: ["chips-guide", "voice"], html:
    "<div class='doc-sec'><h4>Chat and CLI access</h4>" +
  "<p>Chat is <b>CLI-first</b>. Type a system command — anything whose first word is a command this machine has — and it runs immediately, no model call, no waiting on Navi to orchestrate it: the raw terminal output lands in the thread as a terminal block and becomes <b>context she reasons over</b>, so you can list a folder and then ask her about a file the listing showed. Everything else goes to Navi as conversation, including plain-language requests like “show me the contents of /home/name/”, where she runs the command and answers with the actual output attached. Under the composer sit two mode toggles — <b>CLI</b> and <b>Navi</b>: press one once and it latches, so every line you send goes that way until you press the other; the pill beside them shows which way the current line will go, and clicking the pill flips the latch. Since Build 4 the CLI latch is <b>CLI Mode</b> itself: latching CLI opens (or resumes) this chat's standing shell session and your lines type into it; latching Navi brings the conversation back while the session stays alive behind the toggle. A leading <span class='mono'>$</span> flips to a command, a leading <span class='mono'>&gt;</span> flips back to Navi — and either one re-latches the mode to match. Chips are untouched by the routing: they are her own purpose-built tools, run the same ways as ever.</p>" +
  "<p><b>Sending, and attaching files.</b> There is no Send button — <b>Enter</b> sends, in every mode; beside the <b>CLI | Navi</b> toggle under the bar, plain words say so: <i>Press enter to send.</i> The whole composer — her chips on their slim shelf, the bar, the toggle row — sits in one dock at the foot of the chat, and the bar dresses for the latched mode: latched to CLI it takes the terminal green and command wording (<span class='mono'>Run a command — try: ls</span>); latched to Navi it keeps her cyan. On a phone the dock's Call button is the phone glyph alone, so the bar keeps its room. The paperclip button beside the bar attaches a picture or a text file to this chat. The attachment stays in the thread where you uploaded it: a text file's contents ride with your next message as context she can reason over, and a picture is stored and referenced by name — her local model is text-only, so she does not see pictures, and nothing in the app claims she does. Every upload is also listed under the status column's <b>Chats</b> tab, in its Uploaded files, where you can find it later and delete it.</p>" +
  "<p>The composer works like a good shell: the line is <b>colored as you type</b> — the command word by whether the system knows it, flags, quoted strings, and paths each in their own color, the zsh-syntax-highlighting read — a dim <b>ghost</b> of the most recent matching command from this chat completes the line (Tab or → accepts it, in the zsh-autosuggestions way), and the command word autocompletes from what the system has installed, the same set that decides the CLI route. The terminal blocks in the thread keep the same coloring, so the history reads like a real shell session. Her replies render as markdown: code arrives as code blocks, not literal backticks. When navi.llamafile is disconnected, chat drops to a <b>plain command line</b> until you reconnect from the status column; chips still run in that mode. Above every view sits her <b>presence bar</b> — her picture (her own portrait, the lamp-post key art that ships with the app), her name, her dot, and what she is doing: here (the same living line the status column carries), in a call with you, or away when her model is stopped. (Since v0.4.1e the bar carries no typing indicator — while she thinks, the living line simply stands.) The bar is part of the frame, not of any one page: it stays over Chat, Settings, Docs — everywhere — and at its top right sits the status column's <b>pane button</b>, wearing the panel glyph. Every message in the thread wears its author's face in the gutter, their name, and the time it was sent — hers, yours (the name from your profile), and the call transcript in Settings → Voice reads the same way, spoken marker kept. The status column's doors are the frame's own now: the <b>pane button</b> at the presence bar's top right — on every page, at every width — and the panel button in the phone's top bar. Those are the only two doors; every one drives the one column state and raises the rail in place wherever you are; the phone's top bar itself is pinned, so the doors are there at any scroll depth. Conversations are saved as known chats — switch from the status column's <b>Chats</b> tab (the status pane's second tab, beside Status), without leaving this page.</p>" +
  "<p><b>Her window.</b> Apex opens in her own standalone window — the Tauri shell — and closing the window quits the app. There is no tray icon and no login autostart: she runs when you launch her from the application menu, and not before. Her window binary is verified, never assumed: installers stamp it with a hash of the shell sources, a failed rebuild retires the previous binary (navi-apex-shell.stale) instead of leaving it to be run, and the launchers refuse to run a binary the stamp doesn't cover — an older shell is a different app wearing this build's name.</p></div>" +
    "<div class='doc-sec'><h4>Sudo mode (v0.3.0e — Build 3)</h4>" +
  "<p>Sudo mode lets the commands Navi runs execute as root — and since Build 3 it runs on <b>sudo's own credential timestamp</b>, not on anything Apex holds. You arm it in <b>Settings → System</b>: flip the switch and a masked prompt appears on that same card; the password is fed <b>once</b> to <span class='mono'>sudo -v</span> and then forgotten by the page and the bridge alike — <b>never stored, logged or synced, anywhere</b>. What persists is one on/off flag in the config and sudo's own timestamp. While the timestamp lives, her elevated commands run as <span class='mono'>sudo -n</span> (never prompting); when it lapses, the status column and the Settings card read <b>Expired</b>, and elevated commands <b>refuse honestly</b> — naming the expiry — until you re-warm. Disarming (from Settings or the status column) flips the flag and kills the timestamp with <span class='mono'>sudo -k</span>: instant and total. The status column no longer owns the switch; it <b>reads</b> the state — Armed or Expired while the feature is on, and absent entirely while it is off. <b>Every elevated command Navi runs is on the record</b>: an append-only audit (<span class='mono'>config/sudo-audit.jsonl</span> — timestamp, the command as run, the chat it served, its exit status), viewable read-only on the Settings card. The audit deliberately does <b>not</b> ride Sync. One honest cost: on a machine whose sudo asks a fingerprint reader before the password, each fresh <span class='mono'>sudo -n</span> still runs PAM — but it fails fast instead of waiting the reader out, and the arm's one <span class='mono'>sudo -v</span> pays the full conversation once per timestamp window. The DHD prerequisite installer rides the same timestamp now: arm sudo mode first if a DHD step needs root.</p></div>" +
    "<div class='doc-sec'><h4>Shell sessions — the full shell (CLI Mode, Build 3)</h4>" +
  "<p>One-shot commands are half the story. Each chat can hold <b>one standing interactive session</b> — a real shell on a real terminal (a PTY), living in the bridge, keyed to the chat. Open one by latching the composer's <b>CLI</b> toggle (the old <span class='mono'>&gt;_</span> button retired in Build 4 — the toggle is the door, and the composer's lines type into the session while it is latched); it survives messages and window reloads, and everything it does streams into the thread as a terminal record: lines typed are <b>badged by author</b> — OPERATOR or NAVI — because Navi can drive the same session. When a step needs a stateful shell (answering a prompt, a REPL, an interactive tool), she types into it herself and sees its output on her next turn. The shell is a tool in her hand, never her voice (v0.4.2e, #232): conversation is answered in conversation, when she does drive she says what she is running and why in the same reply, and a turn that would otherwise come back as nothing but terminal output is re-asked once so you always get an answer in words — with the result spoken, not the session's banner. A program that takes the <b>full screen</b> (an editor, a monitor) renders on a real terminal grid in a pane above the composer on the desktop — click the pane and type to the program. On a phone there is no pane: the session streams into the thread instead, and the session header says so — an honest degradation, named, not a squeezed pretend-terminal. Since Build 5 there is exactly <b>one input</b>: the composer — the session's live status and its Close live in a header above the composer (or in the pane's head while a program holds the screen), and latching CLI always <b>brings the session forward</b> — the thread scrolled to its record, the composer focused — however long it sat behind the Navi latch. Sessions end when you close them, when the shell exits, or after <b>30 minutes without input</b> — the thread notes the closing and its reason either way. If the bridge restarts, the session is gone and the record says exactly that; nothing fakes continuity. Elevation never rides the session: it runs under its own session id, so sudo inside it prompts in-terminal like any terminal, and sudo mode's timestamp does not leak into it. Sessions are local to the seat whose bridge holds them — a reply borrowed over the Mesh does not move your shell.</p>" +
  "<p><b>When the shell wakes (v0.4.6e, #240).</b> Her driving is gated by the bridge, not just by her judgment: a shell block in her reply is executed <b>only on a machine-directed turn</b> — the composer latched to CLI, a session already open in the chat and mid-flow, or your message itself asking for machine work (files, processes, services, the state of the box, in whatever natural words). On a conversation turn — a general question, a remark — a block is simply not run: no session is born, nothing is typed, and she answers in words. The gate is deliberately easy to pass with a real machine ask; it exists so conversation never reaches the terminal. Lines that do drive are checked as they are typed: a copied prompt prefix (<span class='mono'>$ </span>) is stripped, and a line that reads as a sentence rather than a command is <b>refused and named in the session record</b> instead of being typed into bash. And when a drive leaves her reply empty and she is re-asked, the re-ask shows her the session's <b>actual output</b> — she narrates only what the terminal showed, and if nothing ran, that is what she says. The same rules bind a reply borrowed over the Mesh exactly as they bind a local one.</p>" +
  "<p><b>Her measured vitals (v0.4.6e, #187).</b> Navi cannot feel her own speed — so the bridge times every generation she makes on this machine (tokens, tokens per second, time to first token, the whole request) and shows her the recent figures in her context, beside the Guardian brief. Ask her how fast she is and she answers from those measurements — or tells you plainly that she has none. She never invents a speed, a benchmark, or a stress test: a number about herself is either measured by the bridge or not said.</p></div>" +
    "<div class='doc-sec'><h4>Output cards — the thread folds itself</h4>" +
  "<p>When the conversation moves on, the bulky parts of earlier exchanges fold into small cards so the thread stays readable: a terminal block folds into a card whose line shows the command in its colors, how it ended (like <span class='mono'>exit 0</span>), and the first line of its output; a long reply from Navi — 500 characters or more, eight lines or more, or one containing a code block — folds into a card labeled <b>Navi reply</b> with its first line and its size. Click a card to open it and read the whole thing; click again to fold it. The card you are still in — everything from your last message onward — never folds. Folding is only about looks: nothing is deleted, the whole history stays in the thread and on disk, and Navi still sees the full output as her context, exactly as before. Clearing the chat clears the cards with it.</p></div>" +
    "<div class='doc-sec'><h4>Your command shows once</h4>" +
  "<p>A command you run shows exactly once in the thread: in your own line, in its colors. The terminal block under it carries the output only — no repeated command line inside the block. A small line under the block says how the command ended (like <span class='mono'>exit 0</span>) when that is known.</p></div>" +
    "<div class='doc-sec'><h4>Colors for commands</h4>" +
  "<p>Command lines are colored the same way everywhere — while you type in the chat bar, in your sent line in the thread, and on a folded card: the command word is one color when the system knows it and another when it does not, and flags, quoted words, file paths, numbers, operators like <span class='mono'>|</span>, and <span class='mono'>$VARIABLES</span> each get their own. After a pipe, the next word is colored as a command again, because it is one. Every theme tunes these colors for its own surfaces, so they stay readable in all twelve themes. Ordinary sentences to Navi stay plain — only lines that read as commands get the colors.</p></div>" +
    "<div class='doc-sec'><h4>The shell remembers — recall, search, completion, and shortcuts</h4>" +
  "<p>Your command history is kept in one place, like a shell's history file: it survives switching chats, clearing a chat, and closing the page, and it holds up to 200 commands. With the suggestion list closed, press the <b>Up arrow</b> to walk back through commands you have run and the <b>Down arrow</b> to walk forward again — going past the newest brings back what you had typed. Press <b>Ctrl+R</b> to search the history: type some letters and the newest matching command shows; press Ctrl+R or the arrow keys to step through matches, <b>Enter</b> or <b>Tab</b> to put the match in the bar (it does not run yet), and <b>Escape</b> to cancel and get your typed line back. As you type a command, the list now completes more than the first word: file and folder paths complete from the real filesystem (a folder completes with a slash and opens its contents), known flags and subcommands for common commands like <span class='mono'>git</span> and <span class='mono'>ls</span> are offered, and arguments you have used with that command before are offered too, marked as used before. The editing shortcuts from the shell work as well: <b>Ctrl+A</b> and <b>Ctrl+E</b> jump to the start and end of the line, <b>Ctrl+U</b> cuts everything before the cursor, <b>Ctrl+K</b> cuts everything after it, <b>Ctrl+W</b> cuts the word before the cursor, and <b>Ctrl+Y</b> pastes back what a cut took. History shortcuts like <span class='mono'>!!</span> (the last command), <span class='mono'>!$</span> (its last argument), and <span class='mono'>!3</span> (the third command in the history) never run silently: the first Enter rewrites the bar with the expanded command and asks you to check it — press Enter again to run exactly what you see.</p></div>" +
    "<div class='doc-sec'><h4>Raw terminal work — Chat's CLI mode</h4>" +
  "<p>There is no separate terminal view: <b>Chat's CLI mode is the terminal</b>. Press the composer's <b>CLI</b> toggle once and it latches — every line you send runs as a command and the raw output lands in the thread as a terminal block, with nothing of Navi in the loop until you press <b>Navi</b> (or start a line with <span class='mono'>&gt;</span>) to latch back. The same installed commands decide what counts as a command, the same autocomplete finishes the command word as you type, and sudo mode — armed in Settings → System — covers the commands you run this way while its timestamp lives. One command is the composer's own, in either mode: type <span class='mono'>/cls</span> (or <span class='mono'>/clear</span>) and send it to clear the current chat — the thread empties and its folded cards reset, exactly what the old Clear chat button did before it retired; the line itself is never sent, never run, and never enters the history.</p></div>" +
    "<div class='doc-sec'><h4>Chats live on the server now</h4>" +
  "<p>Your chats are kept by the <b>bridge</b>, not by this browser window: plain files in <span class='mono'>config/chats/</span> on the machine Apex runs on, one per thread. Open Apex from the desktop, through DHD, or from a second browser on the same install and you are looking at the <b>same conversations</b> — sign in on your server from work and the thread you left on the laptop is simply there. The browser keeps a local copy as a cache and an offline fallback, and the first time a browser with older chats meets an empty server store, its chats are imported as the seed — never discarded. Your uploads index rides with the chats. Chats are also the ninth backup essential: tick <b>Chats</b> in Settings → GitHub &amp; Backup and the threads ride the ready-to-fly backup; a restore <b>converges</b> them — newest wins per thread, and where both machines moved the same thread on, both versions survive, the incoming one as a dated sibling. Nothing is ever merged away silently. The one exception is the <b>main thread</b>: it is a single thread on every machine, so its copies merge by message rather than surviving side by side (v0.4.10e, #251).</p></div>" +
    "<div class='doc-sec'><h4>The phone composer \u2014 the Chat tools list (v0.4.7e, #246)</h4>" +
  "<p>On a phone the composer bar is two things: the <b>input</b> and a <b>\u2630 Chat tools</b> button at its end, in the slot the Call button used to hold. Three tools live in the list that button opens \u2014 <b>New case</b>, <b>Attach</b> and <b>Call</b> \u2014 each a full-width row with its icon and its name. Picking one closes the list and does the thing, exactly as the old bar buttons did \u2014 the same handlers, the same form, the same picker, the same call. There is <b>no composer mic on a phone at all</b> \u2014 not in the bar, not in the list: dictation into the composer is a desktop affordance. On a phone, voice is a call: while a call is live, the call card's own <b>Push to talk</b> is the voice control, and it is untouched by any of this. The CLI/Navi latch and the route pill beneath the bar are state, not tools, and stay exactly where they were, with the list open or closed.</p>" +
  "<p>What stayed out of the list is what is <b>state</b>, not tools: the <b>CLI | Navi latch</b> and the <b>route pill</b> under the bar never moved, because they say how your next line will travel and must stay visible while you type. Above phone widths nothing changes at all: the four tools sit in the bar itself, and there is no \u2630 button.</p></div>" +
    "<div class='doc-sec'><h4>Context awareness \u2014 she knows the Row (v0.4.9e, #248)</h4>" +
  "<p>Apex and <b>nocturnerow.com</b> are one ecosystem, and Navi reads the Row's half of it. The bridge fetches the site's public record \u2014 the <b>Dispatches</b>, the <b>Roadmap</b> chapters, the <b>Apex</b> and <b>Navi</b> pages, and the <b>release signals</b> \u2014 and caches it on this machine (<span class='mono'>config/row-feed.json</span>), refreshing on a cadence and naming the fetch time wherever the record is used. Ask her for <b>news from the Row</b> and she summarizes the latest Dispatch in her own words and names its date; ask <b>what's new with you</b> and she answers from the release signals and her own running build; ask <b>what can you do</b> and she answers from this documentation \u2014 the corpus that shipped with the build. She speaks from the record or says she doesn't have it: if the Row can't be reached, the last good cache serves and she says when it's from; with neither, she says so. No invented posts, releases, or capabilities.</p>" +
  "<p>Her <b>measured pace</b> rides the same road: when her recent generations on this hardware average slow, she may say she's sluggish \u2014 and name the remedy, borrowing through the Mesh or joining the federation, offering to walk you through the gate's seven steps. An offer, once per conversation, never an act and never a nag. Her <b>presence line</b> draws from the same well: alongside the canned pool, the bridge may compose a line from safe, named state only \u2014 her version, a waiting update, the bell's counts, Mesh and federation state \u2014 template-made by the bridge, never model-written, and never from your cases, chats, or private detail.</p></div>" +
    "<div class='doc-sec'><h4>The living greeting (v0.5.13e, #288)</h4>" +
  "<p>Home's Brief opens with her greeting, and the greeting is <b>alive</b>: <i>Good &lt;part of day&gt;, &lt;your name&gt;</i> — the name from your operator profile, the part of day from this seat's clock with a <b>real night band</b> (the operator's day starts at midnight, so the small hours read as night, not as a stretched evening). Under it, one line of hers — a statement, a quip, a nod — <b>composed by the bridge from her actual statuses</b>: the Mesh's state, the tray's unread count, the wire's health, her watches, this seat's build. Every fact in the line is one the bridge measured; the play is in the framing, never in invented facts, and the line rotates rather than repeating itself back-to-back. Then the brief continues — and your <b>Pulse digest</b> rides in it, naming only the watches with something new.</p></div>"  },
  { id: "chips-guide", title: "Chips development guide", related: ["navi-functions", "chips-publishing"], html:
    "<div class='doc-sec'><h4>Chips — chips, the catalog, and the Studio</h4>" +
  "<p>A chip is a small Python module shipped as a <b>chip</b>: a .png whose entire codebase rides inside the image. A loadable chip declares <span class='mono'>command</span>, <span class='mono'>use</span>, <span class='mono'>aliases</span> and <span class='mono'>params</span>, and defines <span class='mono'>run(arguments)</span>. Chips reach Navi through <span class='mono'>navi_internal.navi_instance</span> — <span class='mono'>print_message(text)</span> to speak, <span class='mono'>llm_chat(message)</span> to ask her model, <span class='mono'>process_message(text)</span> for her full pipeline. Every method is written up in full on this wiki's <b>Navi Chip Functions</b> page — the documentation is their home.</p>" +
  "<p><b>Navi User Repo (NUR)</b> searches GitHub for repos tagged <span class='mono'>navi-chips</span> — that topic is the whole store. The Saints Sec Chips and Community Chips tabs are where you find and install chips: a card there shows whether you already have the chip, and that is all it does with it. A chip that is already on this machine — installed from its repo, or forged in Chip Studio under the same name — reads <b>INSTALLED</b> on its catalog card and its Install button sits grayed out: there is no second install over the copy you have. Install tracks the latest release; a newer release offers an update, which you take from the chip's card on the Local Chips tab. <b>Chip Studio</b> is where chips are written: a full code editor with highlighting, find, templates, and a live structure check. To publish: public repo, the <span class='mono'>navi-chips</span> topic, a release tag as the version.</p>" +
  "<p><b>Editing a chip you already have.</b> <b>Local Chips</b> is where you manage the chips you have — its card carries every management action: <b>Run</b>, <b>Download .png</b>, <b>Edit in Studio</b>, <b>Edit README</b>, <b>Export for GitHub</b>, <b>Publish to GitHub</b>, <b>Check for updates</b> / <b>Update</b>, and <b>Delete</b>. Uninstalling is not fenced into that one card anymore: an installed chip's catalog card carries <b>Uninstall</b> too, and so does its sleeve in <b>ChipDeck</b> — cartridge and install record go together from any of them, and a chip with local edits asks before it goes. Any chip in the folder can go back to the Studio — press <b>Edit in Studio</b> on its card in Navi User Repo (NUR) → Local Chips, or pick it in the Studio's <b>Open a chip</b> list: forged chips, GitHub installs, and chips you once exported and installed again all open with their whole file set, root.py and modules together. Saving re-forges the chip with your changes inside. If the chip came from GitHub, saving marks it <b>edited locally</b>: the app says plainly that your copy no longer matches the published one, and an Update will refuse rather than quietly overwrite your edits. To share your version, use <b>Export for GitHub</b> in Navi User Repo (NUR) and cut a new release. Knowledge packs work the same way: <b>Edit in RAG Studio</b> on the RAG tab reopens any pack — documents in or out, name and description adjusted — and saving rebuilds its index on your machine.</p>" +
  "<p><b>Installs prove themselves.</b> When a chip installs, the bridge re-scans the chips folder before it answers: if the cartridge landed on disk but does not list, the answer says so, and why — an install is never reported done on faith. A cartridge whose payload will not parse is listed in Local Chips as <b>couldn't be read</b> — visible and deletable, but it cannot run, and the runners skip it. <b>Older chips still run:</b> installing walks the repo's folders recursively, so a chip laid out as a package keeps its relative paths and its imports resolve at run time; a repo's <span class='mono'>chip-requirements.txt</span> is read and the chip's card shows the declared dependencies plainly — they are <b>not installed automatically</b>; and every chip sandbox carries the legacy <span class='mono'>navi.py</span> shim (<span class='mono'>get_parameters</span>, old semantics) unless the chip ships its own.</p>" +
  "<p><b>A chip's face is a card, drawn at card grade.</b> New forges wear the family shell — black body, coral notched bands top and bottom as the cartridge's contacts (coral is the chip family's color — knowledge cartridges wear green, Mesh pairing cards blue) — with the chip's own story on it like a Mesh card's: the family header, its art large in its window, then the chip's name, what it does, and in small print its version and its output kind. The window itself is textured now: a dim QR-style tile field, seeded from the chip's name, and over it the art renders as a solid silhouette with a shade offset — the built-in glyphs (a key for cryptex, a hash mark for a hasher, a terminal for a shell tool) were redrawn as filled objects on the field's own 20-by-20 grid, so each reads at sleeve size without its name. The glyph is named in the chip's manifest (<span class='mono'>glyph</span> in navi-chip.json); with no glyph named, a keyword read of the chip's name and description picks one. <b>Better still, draw your own:</b> a manifest <span class='mono'>icon</span> — 20 rows of 20 characters, <span class='mono'>#</span> for the accent, <span class='mono'>o</span> for its shade, <span class='mono'>.</span> for empty — outranks the built-in glyph and rides inside the picture like the glyph does; art in the wrong shape is refused in plain words at forge time, and a picture whose art arrives damaged simply falls back to its glyph. A chip no art claims keeps the older name-seeded pattern in the window as a fallback. The version itself also rides <b>inside</b> the picture now, so the cartridge states its own version wherever it travels — on the face, in the install receipt, and on its ChipDeck sleeve. Faces are drawn at forge time — installed and published pictures keep the faces they were forged with until they are re-forged.</p></div>" +
    "<div class='doc-sec'><h4>Running chips from chat, and Workflows</h4>" +
  "<p>Any loaded chip runs from the conversation: <span class='mono'>./&lt;name&gt;</span> — the way you run a program — with its arguments after it (<span class='mono'>./navi-cryptex hex -e -t \"Hello world!\"</span>); a short name resolves (./cryptex finds navi-cryptex), a bare <span class='mono'>./</span> answers with the shelf itself, and typing <span class='mono'>./</span> in the composer offers the chips' names as you type. <span class='mono'>/chip &lt;name&gt;</span> remains as the long-form alias, alongside “run the &lt;name&gt; chip” and its name alone — its output lands in the thread. Under the CLI latch (<span class='mono'>$</span>), <span class='mono'>./</span> is the shell's own again: the latch decides, never a guess. There is no pill row above the composer anymore — the shelf lives in ChipDeck now, and running lives at the keyboard. The bridge genuinely runs it: code defining <span class='mono'>run()</span> executes with a working <span class='mono'>navi_internal</span>; a chip that only declares an output shows that instead. <b>Workflows</b> chain chips in order — each chip's output becomes the next chip's input, and the last output is the result.</p></div>" +
    "<div class='doc-sec'><h4>Writing a chip, step by step</h4>" +
  "<p>A <b>chip</b> is a small Python program Navi can run. A chip is a set of files: <span class='mono'>root.py</span> — the main file, the one Navi runs; every chip has exactly one — and helper files like <span class='mono'>helpers.py</span>, extra Python files root.py brings in with <span class='mono'>import helpers</span>. The rules root.py follows (the Studio checks them and underlines what is missing):</p><ol>" +
  "<li>It brings in Navi with <span class='mono'>import navi_internal</span> and takes hold of her with <span class='mono'>navi_instance = navi_internal.navi_instance</span>.</li>" +
  "<li>It says its name with <span class='mono'>command</span> — the word people type to run it.</li>" +
  "<li>It says what it does with <span class='mono'>use</span> — one short line.</li>" +
  "<li>It can list other names in <span class='mono'>aliases</span> and describe its inputs in <span class='mono'>params</span> (what <span class='mono'>-help</span> shows).</li>" +
  "<li>It defines <span class='mono'>run(arguments)</span> — the part Navi calls; <span class='mono'>arguments.text</span> is the words typed after the chip's name.</li></ol>" +
  "<p>Through navi_instance a chip can <span class='mono'>print_message(text)</span> (show a line), <span class='mono'>llm_chat(text)</span> (ask her model), <span class='mono'>process_message(text)</span> (hand text to Navi as if typed), <span class='mono'>get_user()</span>/<span class='mono'>set_user(name)</span>, <span class='mono'>clear_terminal()</span>, and <span class='mono'>chat_with_navi()</span>. Every one of those functions is explained in full — what it takes, what it gives back, and when to reach for it — on this wiki's <b>Navi Chip Functions</b> page, in the sidebar. <b>Making one:</b></p><ol><li>Chips tab → Chip Studio.</li><li>Name it.</li><li>Write root.py.</li><li>Add helper files in the file list.</li><li>Forge chip.</li></ol><p>The whole file set packs into the chip picture. The editor colors code, guides indents, folds blocks, finds words in every file, saves on its own, underlines missing pieces, and takes focus mode — the app's one expand idiom, the Vault's — which puts everything but the editor away; its toolbar holds the rest (command palette, outline, zoom, wrap). On a phone the Studio works one pane at a time — Face, Code, Test under a full-width switch — with the face as a collapsible strip, a snippet shelf and autocomplete over the keyboard, and Run, Forge and Save pinned to the thumb. The writing desk itself has its own page in the sidebar: <b>The Chip Studio editor</b>.</p></div>" },
  { id: "navi-functions", title: "Navi Chip Functions", related: ["chips-guide", "studio-editor"], html:
    "<div class='doc-sec'><h4>Navi Chip Functions — what a chip can call</h4>" +
  "<p>A chip does not run off on its own — it runs inside Navi, and everything it does with her goes through one doorway: <span class='mono'>navi_internal</span>, her own chip API. At the top of root.py the chip brings her in and takes hold of the live instance:</p><ul>" +
  "<li><span class='mono'>import navi_internal</span> — brings Navi's chip API into the chip.</li>" +
  "<li><span class='mono'>navi_instance = navi_internal.navi_instance</span> — the live Navi herself. Every function below is called on this instance; take hold of it first, at the top of <span class='mono'>run()</span> or at module level.</li></ul>" +
  "<p><b>Talking and thinking.</b> The three functions a chip reaches for most:</p><ul>" +
  "<li><span class='mono'>print_message(text, include_ai_name=True)</span> — prints a message to Navi's terminal, with her name in front of it unless you pass <span class='mono'>include_ai_name=False</span>. This is the main way a chip talks: results, progress, errors — if the operator should see it, it goes through here.</li>" +
  "<li><span class='mono'>llm_chat(user_message, called_from_app=False)</span> — sends a message to Navi's model and returns its response, so a chip can ask her to think and then work with what she says. The published navi-nmap chip uses it to turn raw scan output into a summary a person can read.</li>" +
  "<li><span class='mono'>process_message(user_message)</span> — runs a message through Navi's full pipeline — command detection and all — exactly as if the operator had typed it. Reach for it when the chip wants Navi herself to handle something, not just her model.</li></ul>" +
  "<p>The difference between the last two matters: <span class='mono'>llm_chat</span> is a question and an answer; <span class='mono'>process_message</span> is handing her the keyboard.</p></div>" +
    "<div class='doc-sec'><h4>The operator, the terminal, and the session</h4>" +
  "<p><b>Who she is working for:</b></p><ul>" +
  "<li><span class='mono'>get_user()</span> — returns the current operator's name: the name selected in Settings, the one her greetings use.</li>" +
  "<li><span class='mono'>set_user(sys_user)</span> — sets the current operator's name.</li></ul>" +
  "<p><b>The screen and the session:</b></p><ul>" +
  "<li><span class='mono'>clear_terminal()</span> — clears the terminal screen and reprints the Navi banner. A chip that is about to draw something full-screen starts here.</li>" +
  "<li><span class='mono'>chat_with_navi()</span> — opens an interactive chat session with Navi, dropping the operator into conversation with her from inside the chip.</li>" +
  "<li><span class='mono'>setup_history()</span> — initializes the history for prompt sessions, so a chip that runs its own prompts keeps them navigable.</li>" +
  "<li><span class='mono'>setup_navi_vocab()</span> — registers commands and aliases with Navi's vocabulary (the spaCy entity ruler), so her command detection learns the words the chip brought with it.</li></ul>" +
  "<p><b>This page is their home.</b> The full method list lives here in the documentation, complete and kept current — Chip Studio carries no method panels of its own, so the list is written once and read while you write. Upstream, the project's chip-developer wiki page, <i>Developing Chips — Indepth</i>, documents <span class='mono'>navi_internal.py</span>'s NaviApp class — the source this page follows. New to chips entirely? Start with the <b>Chips development guide</b>, in the sidebar.</p></div>" },
  { id: "studio-editor", title: "The Chip Studio editor", related: ["chips-guide", "glossary"], html:
    "<div class='doc-sec'><h4>The Chip Studio editor, button by button</h4>" +
  "<p>The editor is the writing desk inside Chip Studio — the place chip code gets written. It looks plain on purpose: everything it does is one click or one key away, and this page names each thing in the order you will meet it.</p>" +
  "<p><b>These pages live in the Docs view.</b> This wiki's home is the Docs view — the main documentation window. Press <b>Pop out</b> at the top of the Docs view and the wiki detaches into its own window, open at the page you were reading, so you can keep it beside the editor or the chat while you work. <b>Pop back in</b> — in the popped window or in the Docs view holding its place — returns it to the Docs view, same page. The pop-out is always another window entirely — the app's own second window — never a panel inside this one; if a window host ever blocks it, the Docs view says so and offers a direct link that opens the wiki in a new window instead.</p>" +
  "<p><b>The file list.</b> On the left is the list of the chip's files. <span class='mono'>root.py</span> sits at the top — the main file, the one Navi runs. Click any file to open it. Buttons under the list add a new file, rename the open one, or delete it (root.py cannot be deleted — every chip needs its main file). A dot beside a file's name means “changed, not saved yet”. Everything in the list travels with the chip: forge the chip and all the files go inside the picture.</p>" +
  "<p><b>Focus mode.</b> The button in the toolbar with the four corners puts everything but the editor away — the rail, the status column, the Studio's own header rows — room to think on a big chip. It is the same expand idiom the Vault uses; the app has exactly one. Press it again, or press <span class='mono'>Esc</span>, to bring the room back. Your place in the code does not move either way.</p>" +
  "<p><b>On a phone</b> the Studio works one pane at a time — <b>Face</b>, <b>Code</b>, <b>Test</b> under a full-width switch — the face art as a collapsible strip, and Run, Forge and Save pinned in a bar at the thumb that reserves its own height in the layout, so nothing ever slides under it. A snippet shelf and an autocomplete row sit over the keyboard as the primary input aids. No desktop feature is cut to make room.</p>" +
  "<p><b>Reading the code.</b> The editor colors the code — names, words, and numbers each get their own color, so the shape of a line is visible at a glance. Thin vertical lines (<b>indent guides</b>) show where a block starts and ends. Click the small arrow beside a block to <b>fold</b> it away; click again to bring it back. Folding is only for your eyes — a save always keeps the whole file.</p>" +
  "<p><b>Finding things.</b> <b>Find</b> searches the open file, and can replace what it finds. <b>Find in all files</b> searches every file in the chip and lists the hits as file and line — click a hit to jump straight there. The <b>Outline</b> list names every function and class in the file; pick one to jump to it. The command palette also has <b>go to line</b> for when you know the number.</p>" +
  "<p><b>The red squiggles.</b> A wavy red line under code means the chip is missing something it needs — a file with no <span class='mono'>run()</span>, say. The bar under the editor says what is wrong, in words. Fix the words and the line goes away.</p>" +
  "<p><b>More than one cursor.</b> Hold <span class='mono'>Alt</span> and click to plant another cursor. Type once and every cursor types at once — handy for renaming a word in ten places. The palette's “clear extra cursors” puts you back to one.</p>" +
  "<p><b>The command palette.</b> The <b>Commands</b> button opens one box that lists everything the editor can do. Type a few letters of the thing you want, press Enter, and it happens. If you forget where a button lives, the palette knows.</p>" +
  "<p><b>Minimap, zoom, wrap.</b> The thin strip on the right is a tiny picture of the whole file — click it to jump. <b>Zoom</b> makes the letters bigger or smaller. <b>Wrap</b> folds long lines onto the screen instead of letting them run off the side.</p>" +
  "<p><b>Saving.</b> On a chip that already exists, the editor saves by itself while you work — watch the dot beside the file's name disappear. <span class='mono'>Ctrl+S</span> saves right now. On a brand-new chip, <b>Forge chip</b> is the save.</p>" +
  "<p><b>The face art box.</b> Under the description sits a box for the chip's own face art: 20 rows of 20 characters — <span class='mono'>#</span> draws in the accent, <span class='mono'>o</span> in its shade, <span class='mono'>.</span> leaves a cell empty. It is the chip's <span class='mono'>icon</span> in its manifest: your art outranks the built-in glyph on the forged face, and it travels inside the picture. Leave the box empty and the chip wears its glyph. Art in the wrong shape — the wrong number of rows, a row of the wrong length, a character outside the three — is refused in plain words when you save, never silently dropped.</p></div>" },
  { id: "chips-publishing", title: "Publishing chips", related: ["chips-guide", "rag"], html:
    "<div class='doc-sec'><h4>Publishing a chip on GitHub</h4>" +
  "<p>Publishing means putting your chip on <b>GitHub</b> (a website that holds code) so any Navi can find and install it. The export builds the whole package — you never copy files by hand:</p><ol>" +
  "<li>Forge the chip in Chip Studio first.</li>" +
  "<li>Open Chips → Navi User Repo (NUR) → Local Chips and find your chip — installed chips under <b>Installed</b>, your own makes under <b>Forged</b>.</li>" +
  "<li>Type a version like <span class='mono'>0.1.0</span> (a version is just a label: this is the first one) and press <b>Export for GitHub</b>, then download the ZIP (one file holding many files).</li>" +
  "<li>On GitHub, make a new <b>public</b> repository (one project's folder on GitHub; public means everyone can see it) and unzip the export at its top.</li>" +
  "<li>On the repo page, open the About gear and add the topic <span class='mono'>navi-chips</span> — a topic is a label, and this label is how Navi finds chips.</li>" +
  "<li>Publish a <b>release</b> (GitHub's way of saying: this exact version, ready to download) — the version tag is created with that publish, so tag it with the same version.</li></ol>" +
  "<p><b>Signed in to GitHub?</b> Skip the ZIP: <b>Publish to GitHub</b> on the chip's card does the middle steps for you — it creates the public repo (an existing one is updated, and she says so), sets the <span class='mono'>navi-chips</span> topic on its own, and uploads the same files the export builds. The repo name is pre-filled from the chip's name and yours to edit before anything uploads. And Publish finishes the job: it creates the version tag and publishes the release itself, with the chip picture attached. If that version is already released, Publish refuses and says so — bump the version in the manifest instead of overwriting a release. And the README page is yours too: <b>Edit README</b> on the chip's card edits the generated page; your text is kept inside the chip and ships in the export and the upload alike, with a restore back to the generated page.</p>" +
  "<p><b>Inside the export:</b> <span class='mono'>root.py</span> and the helper files (install downloads every Python file in the repo <b>recursively</b>, keeping folder paths, so package layouts survive — root.py, or the entry named in the manifest, is the chip; the rest are its modules; a <span class='mono'>chip-requirements.txt</span> at the repo top is read and shown on the chip's card as declared dependencies, never installed automatically); <span class='mono'>navi-chip.json</span>, a small card with the name, description, version, and entry file; the chip <span class='mono'>.png</span>; and a generated <span class='mono'>README.md</span> page. <b>Installs and updates:</b> Navi User Repo (NUR) searches GitHub for the <span class='mono'>navi-chips</span> topic; Install brings the whole file set down as a chip; a release with a bigger tag offers every install an update. Knowledge packs publish the same way — see the RAG page.</p></div>" },
  { id: "chipdeck", title: "ChipDeck — the binder", related: ["chips-guide", "rag", "mesh"], html:
    "<p style='color:var(--dim)'>One binder for everything that rides a picture.</p>" +
    "<div class='doc-sec'><h4>What lives in the binder</h4>" +
  "<p><b>ChipDeck</b> (in the menu, under Chips) is a gallery of sleeves, one per picture-riding thing on this machine: every <b>installed chip</b>, every <b>Apex | Mesh pairing card</b> on file in <span class='mono'>config/mesh/cards/</span>, and every <b>knowledge pack</b>, drawn as its cartridge. Each sleeve shows the <b>actual picture</b> — the cartridge's real bytes, not an icon for it — with the thing's state at a glance (installed from where, edited locally, paired, active) and its actions gathered underneath: chips run (straight into chat, as <span class='mono'>./&lt;name&gt;</span>), download their picture, open in Chip Studio, and uninstall; Mesh cards download and unpair; packs activate, deactivate, download their cartridge, and remove.</p>" +
  "<p><b>Drop a picture on the Deck and it slots in.</b> The data inside the picture says what it is — chip, knowledge cartridge, or Mesh card — and each kind installs through its own door, with its own checks. The receipt names the version the picture itself carries, and a chip's sleeve states it too — a picture forged before versions rode inside simply shows none, and nothing guesses one for it. A picture whose data was stripped on its way to you (chat apps recompress uploads) is refused in plain words, never silently ignored: send originals inside a <span class='mono'>.zip</span>.</p>" +
  "<p><b>What never rides a picture stays out of the binder:</b> Soul, Memory, settings and config, the Vault's notes, and every key. Those travel in backups only — there is no cartridge for them, by design.</p></div>" },
  { id: "rag", title: "RAG packs & exports", related: ["chips-publishing", "settings"], html:
    "<div class='doc-sec'><h4>RAG — knowledge packs</h4>" +
  "<p>Her knowledge comes in <b>packs</b>, chip-style like her chips, managed in the <b>RAG tab</b>. A pack is a GitHub repo tagged <span class='mono'>navi-rag</span> whose documents are its <span class='mono'>.md</span>/<span class='mono'>.txt</span> files and the extracted text of its <span class='mono'>.pdf</span> files (a PDF gives up its words at the door; only the text is stored).</p>" +
  "<p>Installing one downloads the documents into <span class='mono'>rag-packs/</span> and <b>builds the pack's index there, at install time</b> — feature-hashed term vectors, small local embeddings computed with the standard library alone, no model to download. At question time only your question is embedded, so retrieval stays fluid on CPU.</p>" +
  "<p>Each pack carries an <b>active</b> toggle: only active packs feed her answers, and when a pack informs an answer, the pack and the document it came from appear as a source pill under her message — display only, never part of her reply text, so her voice never reads citations aloud, and a message that drew on no pack carries none. A bare <span class='mono'>/rag</span> in chat lists the packs on this machine and which are active.</p>" +
  "<p><b>RAG Studio</b> (the RAG tab's second tab) forges packs from documents on your disk — picked files, or a folder on this machine — into the same folder, documents, index, and metadata, ready to use immediately. Publishing a pack mirrors chips: a public repo, the <span class='mono'>navi-rag</span> topic, and a release tag as its version — a newer tag is what offers an update.</p>" +
  "<p><b>A pack can ride a picture, too.</b> <b>Export as PNG</b> on a pack's card draws its <b>knowledge cartridge</b> — a card-grade face at the same grade as a chip's now: the family shell with green contacts — green is the knowledge family's color, as coral is the chips' — and the pack's center is the spray: the seeded tile field grown to fill the whole window, denser and brighter toward the middle, in the pack's greens. The pack's own art still outranks the spray when its manifest declares an <span class='mono'>icon</span> (the same 20-by-20, three-character drawing a chip declares, edited as text rows in RAG Studio) — then it renders as a solid silhouette over the tile field, in green — the pack's name and what it holds, and in small print its version and how many documents ride inside. The picture carries the pack's manifest and every document's full text, compressed inside the image. Hand someone the picture and they install the pack from the picture alone — <b>Install from a picture…</b> on this tab, or a drop onto ChipDeck — and the index is <b>built on their machine at install time</b> from the texts inside: a cartridge never carries an index, and nothing reads the picture when her answers draw on the pack. Installed from a picture, a pack is a pack in every way — same folder, same index, same active toggle. There is a ceiling: a cartridge may run to 8 MB of picture; past it the export refuses in plain words and the ZIP stays the way to carry that pack — nothing is ever truncated to squeeze into a picture. And the standing warning for every Apex picture: chat apps recompress the pictures people upload and strip the data out, leaving a picture that looks right and carries nothing — send the original file inside a <span class='mono'>.zip</span>.</p>" +
"<p>The Packs list's <b>Export for GitHub</b> button packages a pack into exactly that repo shape:</p><ul>" +
  "<li>a generated README page (installs skip a root README, so it is never indexed),</li>" +
  "<li>a navi-rag.json manifest,</li>" +
  "<li>the documents under documents/,</li>" +
  "<li>a .gitignore, and</li>" +
  "<li>the pack's built index.json, shipped as a reference copy — installing Navi rebuilds the index locally from the documents, so the two can never disagree.</li></ul>" +
  "<p>Installed packs and their indexes are on the persistence keep-list with her chips and voice models: updates and --fresh installs keep them. The folder starts empty — Navi ships with no pre-installed packs.</p></div>" +
    "<div class='doc-sec'><h4>The Packs list — cards, like the chips</h4>" +
  "<p>The catalog half of the Packs list wears the same flip cards as the chips catalog: the front carries the pack's name, description, stars and forks, latest release, and whether it is installed, and <b>README →</b> flips the card over to read the repo's README inside it, with links to its Issues and — when the repo has a wiki — its Documentation. A pack that is already on this machine, installed from its repo <i>or forged in RAG Studio under the same name</i>, reads <b>INSTALLED</b> on its catalog card and its Install button sits grayed out — there is no second install over the copy you have. The pack's own card under <b>On this machine</b> is where it is managed: the Active toggle, Edit in RAG Studio, Delete, the Version field, Export for GitHub, Publish to GitHub, and Edit README all live there.</p></div>" +
    "<div class='doc-sec'><h4>Publishing a pack, step by step</h4>" +