# app/index.html — part 9 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 15698–16438 of app/index.html.

    vqa("[data-vault-menu]").forEach(function (b) { b.onclick = function () { var p = decodeURIComponent(b.getAttribute("data-vault-menu")); vaultMenuPath = vaultMenuPath === p ? null : p; render(); }; });
    vqa("[data-vault-fmenu]").forEach(function (b) { b.onclick = function () { var f = decodeURIComponent(b.getAttribute("data-vault-fmenu")); vaultFolderMenuPath = vaultFolderMenuPath === f ? null : f; render(); }; });
    vqa("[data-vault-rename]").forEach(function (b) { b.onclick = function () { vaultMenuPath = null; vaultRenamePath = decodeURIComponent(b.getAttribute("data-vault-rename")); render(); }; });
    vqa("[data-vault-move]").forEach(function (b) { b.onclick = function () { vaultMenuPath = null; vaultMovePath = decodeURIComponent(b.getAttribute("data-vault-move")); render(); }; });
    vqa("[data-vault-movecancel]").forEach(function (b) { b.onclick = function () { vaultMovePath = null; render(); }; });
    vqa("[data-vault-movego]").forEach(function (b) { b.onclick = function () { var from = decodeURIComponent(b.getAttribute("data-vault-movego")); var sel = document.getElementById("vault-move-sel"); vaultDoMove(from, sel ? sel.value : ""); }; });
    vqa("[data-vault-renamesave]").forEach(function (b) { b.onclick = function () {
      var from = decodeURIComponent(b.getAttribute("data-vault-renamesave"));
      var inp = document.getElementById("vault-rename-in"); var name = inp ? inp.value.trim().replace(/\.md$/i, "") : "";
      vaultRenamePath = null;
      if (!name) { render(); return; }
      var dir = from.indexOf("/") === -1 ? "" : from.slice(0, from.lastIndexOf("/"));
      var to = (dir ? dir + "/" : "") + name + ".md";
      if (to !== from) { vaultSel = to; vaultApply("move", { from: from, to: to }); } else render();
    }; });
    vqa("[data-vault-del]").forEach(function (b) { b.onclick = function () { vaultMenuPath = null; vaultConfirmPath = decodeURIComponent(b.getAttribute("data-vault-del")); render(); }; });
    vqa("[data-vault-delok]").forEach(function (b) { b.onclick = function () { var p = decodeURIComponent(b.getAttribute("data-vault-delok")); vaultConfirmPath = null; vaultApply("del", { path: p }); }; });
    vqa("[data-vault-delcancel]").forEach(function (b) { b.onclick = function () { vaultConfirmPath = null; render(); }; });
    vqa("[data-vault-foldel]").forEach(function (b) { b.onclick = function () { vaultConfirmFolder = decodeURIComponent(b.getAttribute("data-vault-foldel")); render(); }; });
    vqa("[data-vault-foldelok]").forEach(function (b) { b.onclick = function () { var f = decodeURIComponent(b.getAttribute("data-vault-foldelok")); vaultConfirmFolder = null; vaultApply("delfolder", { path: f }); }; });
    vqa("[data-vault-foldelcancel]").forEach(function (b) { b.onclick = function () { vaultConfirmFolder = null; render(); }; });
    vqa("[data-vault-newin]").forEach(function (b) { b.onclick = function () { vaultFolderMenuPath = null; var sel = document.getElementById("vault-new-folder"); if (sel) sel.value = decodeURIComponent(b.getAttribute("data-vault-newin")); var nm = document.getElementById("vault-new-name"); if (nm) nm.focus(); }; });
    vqa("[data-vault-focus]").forEach(function (b) { b.onclick = function () { vaultFocus = !vaultFocus; render(); }; });
    vqa("[data-vault-createmissing]").forEach(function (b) { b.onclick = function () { var t = vaultMissing; vaultMissing = null; if (t) { vaultSel = t + ".md"; vaultApply("save", { path: t + ".md", content: "# " + t + "\n\n" }); } }; });
    vqa("[data-vault-dismissmissing]").forEach(function (b) { b.onclick = function () { vaultMissing = null; render(); }; });
    // Drag and drop (build .73 hardening): the drag source is tracked
    // in vaultDragFrom, every real target (folder headers + the Vault
    // root row) outlines itself the moment a drag starts and fills in
    // under the pointer, and a drop anywhere else — on a note, on
    // empty tree space, on the editor — does nothing at all. Before
    // this round the whole tree sat inside the root drop zone, so a
    // missed drop silently filed the note at the top of the list.
    var vaultClearDropMarks = function () { vqa("[data-vault-drop]").forEach(function (t) { t.style.outline = ""; t.style.background = ""; }); };
    vqa("[data-vault-drag]").forEach(function (el) {
      el.addEventListener("dragstart", function (ev) {
        vaultDragFrom = decodeURIComponent(el.getAttribute("data-vault-drag"));
        el.style.opacity = "0.45";
        if (ev.dataTransfer) { ev.dataTransfer.effectAllowed = "move"; ev.dataTransfer.setData("text/x-vault-path", vaultDragFrom); ev.dataTransfer.setData("text/plain", vaultDragFrom); }
        vqa("[data-vault-drop]").forEach(function (t) { t.style.outline = "1px dashed var(--cy)"; t.style.outlineOffset = "-1px"; });
      });
      el.addEventListener("dragend", function () { vaultDragFrom = null; el.style.opacity = ""; vaultClearDropMarks(); });
    });
    vqa("[data-vault-drop]").forEach(function (el) {
      el.addEventListener("dragover", function (ev) { ev.preventDefault(); ev.stopPropagation(); if (ev.dataTransfer) ev.dataTransfer.dropEffect = "move"; el.style.outline = "2px solid var(--cy)"; el.style.background = "rgba(127,127,127,0.14)"; });
      el.addEventListener("dragleave", function (ev) { if (!el.contains(ev.relatedTarget)) { el.style.outline = vaultDragFrom ? "1px dashed var(--cy)" : ""; el.style.background = ""; } });
      el.addEventListener("drop", function (ev) {
        ev.preventDefault(); ev.stopPropagation();
        var folder = decodeURIComponent(el.getAttribute("data-vault-drop") || "");
        vaultClearDropMarks();
        if (ev.dataTransfer.files && ev.dataTransfer.files.length) { vaultDragFrom = null; vaultImportFiles(ev.dataTransfer.files, folder); return; }
        var from = vaultDragFrom || (ev.dataTransfer ? ev.dataTransfer.getData("text/x-vault-path") : "");
        vaultDragFrom = null;
        if (!from) return;
        vaultDoMove(from, folder);
      });
    });
    if (!mainEl._vaultDropBound) {
      mainEl._vaultDropBound = true;
      mainEl.addEventListener("dragover", function (ev) { if (view === "Vault" && ev.dataTransfer && ev.dataTransfer.types && Array.prototype.indexOf.call(ev.dataTransfer.types, "Files") !== -1) ev.preventDefault(); });
      mainEl.addEventListener("drop", function (ev) { if (view === "Vault" && ev.dataTransfer && ev.dataTransfer.files && ev.dataTransfer.files.length) { ev.preventDefault(); vaultImportFiles(ev.dataTransfer.files, ""); } });
    }
    var searchIn = vq("#vault-search");
    if (searchIn) searchIn.oninput = function () {
      var q = searchIn.value.trim().toLowerCase(); var box = document.getElementById("vault-search-results");
      if (!box || !vaultState) return;
      if (!q) { box.innerHTML = ""; return; }
      var hits = vaultState.notes.filter(function (n) { return n.title.toLowerCase().indexOf(q) !== -1 || n.content.toLowerCase().indexOf(q) !== -1; });
      box.innerHTML = "<div class='card'>" + (hits.length ? hits.map(function (n) { return "<button class='ghost' data-vault-open='" + vaultEnc(n.path) + "'>" + esc(n.path) + "</button>"; }).join(" ") : "<span style='color:var(--dim)'>Nothing in the vault matches.</span>") + "</div>";
      box.querySelectorAll("[data-vault-open]").forEach(function (b) { b.onclick = function () { vaultOpenPath(decodeURIComponent(b.getAttribute("data-vault-open"))); }; });
    };
    var createBtn = vq("#vault-new-create");
    if (createBtn) createBtn.onclick = function () {
      var nm = document.getElementById("vault-new-name"); var name = nm ? nm.value.trim().replace(/\.md$/i, "") : "";
      if (!name) return;
      var fo = document.getElementById("vault-new-folder"); var folder = fo ? fo.value : "";
      var tp = document.getElementById("vault-new-template");
      var content = "# " + name + "\n\n";
      if (tp && tp.value) { var tpl = vaultState.notes.filter(function (n) { return n.path === decodeURIComponent(tp.value); })[0]; if (tpl) content = tpl.content.split("Untitled").join(name); }
      var path = (folder ? folder + "/" : "") + name + ".md";
      vaultSel = path;
      vaultApply("save", { path: path, content: content });
    };
    var folderBtn = vq("#vault-folder-add");
    if (folderBtn) folderBtn.onclick = function () { var fi = document.getElementById("vault-folder-name"); var name = fi ? fi.value.trim() : ""; if (name) vaultApply("folder", { path: name }); };
    var dailyBtn = vq("#vault-daily");
    if (dailyBtn) dailyBtn.onclick = function () {
      var today = new Date().toISOString().slice(0, 10); var path = "Daily/" + today + ".md";
      if (vaultState && vaultState.notes.some(function (n) { return n.path === path; })) vaultOpenPath(path);
      else { vaultSel = path; vaultApply("save", { path: path, content: "# " + today + "\n\n## Today\n\n- \n" }); }
    };
    var exportBtn = vq("#vault-export");
    if (exportBtn) exportBtn.onclick = function () {
      if (CONFIG.backend === "live") { window.open("/api/vault/export", "_blank"); vaultMsg = "Exported the vault as a .zip of the folder — unzip it and Obsidian opens it as-is."; render(); return; }
      var bundle = vaultState.notes.map(function (n) { return "<!-- " + n.path + " -->\n\n" + n.content; }).join("\n\n---\n\n");
      var url = URL.createObjectURL(new Blob([bundle], { type: "text/markdown" }));
      var a = document.createElement("a"); a.href = url; a.download = "apex-vault-export.md"; a.click(); URL.revokeObjectURL(url);
      vaultMsg = "Mock mode exports a single markdown bundle; the live bridge exports the real .zip of the folder.";
      render();
    };
    // The mobile furniture (#279): the New button opens
    // the create sheet; the reader's back button returns to
    // the list with its scroll and the selection intact.
    var newOpen = vq("#vault-new-open");
    if (newOpen) newOpen.onclick = function () { vaultSheetOpen = true; render(); };
    var backBtn = vq("#vault-back");
    if (backBtn) backBtn.onclick = function () {
      vaultReader = false; render();
      window.scrollTo(0, vaultListScroll || 0);
    };
    // The Vault door of the element delete (#280): arm,
    // disarm, fire — the confirm names the blast radius
    // inline, the fire goes to the same bridge act the
    // dossier door uses, with the door marked "vault".
    vqa("[data-vault-el-del]").forEach(function (b) { b.onclick = function () { vaultElArm = b.getAttribute("data-vault-el-del"); render(); }; });
    vqa("[data-vault-el-delno]").forEach(function (b) { b.onclick = function () { vaultElArm = ""; render(); }; });
    vqa("[data-vault-el-delok]").forEach(function (b) { b.onclick = function () {
      var eid = b.getAttribute("data-vault-el-delok");
      vaultElArm = "";
      fetch("/api/dossier", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ action: "delete-element", elementId: eid, from: "vault" }) }).then(function (r) { return r.json(); }).then(function (j) {
        vaultCaseEls = null;
        if (j && j.ok) vaultMsg = "Deleted the element everywhere — the dossier, every case, the Vault.";
        else vaultMsg = (j && j.error) || "The element could not be deleted.";
        refreshVault();
      }).catch(function () { vaultMsg = "The element could not be deleted — the bridge did not answer."; render(); });
    }; });
    // Press-inside opens the editor (v0.5.11e, #284): a
    // press in the rendered note's text swaps the viewer
    // for the editor on the same words — the draft is
    // untouched by the swap. Presses on the note's own
    // working furniture (wikilinks, links, buttons) keep
    // their acts, and a press that ends as a text
    // selection is a read, not an edit.
    var vp = vq("[data-vault-pressedit]");
    if (vp) vp.onclick = function (ev) {
      var t = ev.target;
      if (t.closest && t.closest("a, button, input, select, textarea, label, [data-wikilink]")) return;
      var selTx = window.getSelection ? String(window.getSelection()) : "";
      if (selTx) return;
      var r0 = vp.getBoundingClientRect();
      var frac = r0.height > 0 ? Math.min(1, Math.max(0, ((ev.clientY || 0) - r0.top) / r0.height)) : 0;
      vaultEditing = true; render();
      var ed2 = document.getElementById("vault-ed");
      if (ed2) {
        ed2.focus();
        // Best-effort caret: the pressed point's depth in
        // the rendered note maps to the same depth in the
        // source's lines — the neighborhood, not the letter.
        try {
          var dlines = String(vaultDraft || "").split("\n");
          var li = Math.min(dlines.length - 1, Math.floor(frac * dlines.length));
          var off = 0; for (var di = 0; di < li; di++) off += dlines[di].length + 1;
          ed2.setSelectionRange(off, off);
        } catch (e) {}
      }
    };
    var ed = vq("#vault-ed");
    if (ed) ed.oninput = function () {
      vaultDraft = ed.value;
      var saved = document.getElementById("vault-saved");
      if (saved) saved.textContent = "Edited — autosaving… · " + vaultWordCount(vaultDraft) + " words";
      if (vaultSaveTimer) clearTimeout(vaultSaveTimer);
      vaultSaveTimer = setTimeout(function () {
        var path = vaultSel; var content = vaultDraft;
        if (CONFIG.backend === "live") {
          fetch("/api/vault/save", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ path: path, content: content }) }).then(function () {
            var el2 = document.getElementById("vault-saved");
            if (el2 && vaultSel === path) el2.textContent = "Saved · " + new Date().toLocaleTimeString() + " · " + vaultWordCount(content) + " words";
            vaultState = null; vaultLoad();
          }).catch(function () {});
        } else {
          var st = vaultMockStore();
          st.notes.forEach(function (n) { if (n.path === path) { n.content = content; n.updatedAt = new Date().toISOString(); } });
          vaultMockSave(st); vaultLoad();
          var el3 = document.getElementById("vault-saved");
          if (el3 && vaultSel === path) el3.textContent = "Saved · " + new Date().toLocaleTimeString() + " · " + vaultWordCount(content) + " words";
        }
      }, 900);
    };
  } else if (view === "RAG") {
    mainEl.querySelectorAll("[data-ragtab]").forEach(function (b) { b.onclick = function () { ragTab = b.getAttribute("data-ragtab"); render(); }; });
    // Pack flip cards (issue #23): the same flip discipline as
    // the NUR cards — a click flips, except on a button, a link,
    // a form control, or the README scroll area; the README/Front
    // buttons flip on purpose. The README loads on first flip.
    mainEl.querySelectorAll("[data-ragflipcard]").forEach(function (card) {
      card.onclick = function (ev) {
        var t = ev.target;
        var flipBtn = t.closest ? t.closest("[data-flipbtn]") : null;
        if (!flipBtn && t.closest && t.closest("button, a, input, label, .flip-scroll")) return;
        card.classList.toggle("flipped");
        if (card.classList.contains("flipped")) loadRagReadme(card.getAttribute("data-ragflipcard"));
      };
    });
    var ragRefresh = document.getElementById("rag-refresh");
    if (ragRefresh) ragRefresh.onclick = function () { ragCatalog = null; loadRagCatalog(); };
    var ragPost = function (path, body, done) {
      fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) })
        .then(function (r) { return r.json(); })
        .then(function (j) { done(j); ragCatalog = null; ragPacksState = null; if (view === "RAG") { loadRagCatalog(); refreshRagPacks(true); } })
        .catch(function () { ragMsg = "The bridge didn't answer."; render(); });
    };
    mainEl.querySelectorAll("[data-raginstall]").forEach(function (b) { b.onclick = function () { ragMsg = "Installing " + b.getAttribute("data-raginstall") + " — downloading its documents and building the index…"; render(); ragPost("/api/rag/install", { full_name: b.getAttribute("data-raginstall") }, function (j) { ragMsg = j.ok ? "Installed — the pack is on this machine and active." : (j.error || "Install failed."); }); }; });
    mainEl.querySelectorAll("[data-ragupdate]").forEach(function (b) { b.onclick = function () { ragPost("/api/rag/update", { full_name: b.getAttribute("data-ragupdate") }, function (j) { ragMsg = !j.ok ? (j.error || "Update failed.") : j.updated ? "Updated — documents re-downloaded and the index rebuilt. Its active flag is unchanged." : "Already current — no newer release or push."; }); }; });
    mainEl.querySelectorAll("[data-ragdelete]").forEach(function (b) { b.onclick = function () { ragPost("/api/rag/delete", { full_name: b.getAttribute("data-ragdelete") }, function (j) { ragMsg = j.ok ? "Deleted — the pack's documents and index are gone from this machine." : (j.error || "Delete failed."); }); }; });
    mainEl.querySelectorAll("[data-ragtoggle]").forEach(function (b) { b.onchange = function () { ragPost("/api/rag/toggle", { full_name: b.getAttribute("data-ragtoggle"), active: b.checked }, function (j) { ragMsg = j.ok ? "" : (j.error || "Toggle failed."); }); }; });
    mainEl.querySelectorAll("[data-ragedit]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-ragedit"); ragForgeMsg = "Opening the pack…"; fetch("/api/rag/pack-source?slug=" + encodeURIComponent(slug)).then(function (r) { return r.json(); }).then(function (j) { if (!j.ok) { ragMsg = j.error || "That pack could not be opened."; render(); return; } var meta = (ragPacksState || []).filter(function (p) { return p.slug === slug; })[0]; ragEdit = { slug: slug, sourceKey: meta && meta.source === "github" ? meta.fullName : null }; ragEditPrefill = { name: j.name, description: j.description, icon: (j.icon && j.icon.length) ? j.icon.join("\n") : "" }; ragForgeDocs = j.documents; ragForgeMsg = meta && meta.locallyEdited ? "This pack already carries local edits — it diverges from " + (ragEdit.sourceKey || "its source") + "." : ""; ragTab = "studio"; render(); }).catch(function () { ragMsg = "The bridge didn't answer."; render(); }); }; });
    mainEl.querySelectorAll("[data-rfremove]").forEach(function (b) { b.onclick = function () { var i = Number(b.getAttribute("data-rfremove")); ragForgeDocs = ragForgeDocs.filter(function (_, ix) { return ix !== i; }); render(); }; });
    mainEl.querySelectorAll(".rf-doc-body").forEach(function (t) { t.oninput = function () { var i = Number(t.getAttribute("data-rfdoc")); if (ragForgeDocs[i]) ragForgeDocs[i].body = t.value; }; });
    var rfNew = document.getElementById("rf-new");
    if (rfNew) rfNew.onclick = function () { ragEdit = null; ragEditPrefill = null; ragForgeDocs = []; ragForgeMsg = ""; render(); };
    mainEl.querySelectorAll("[data-ragcartridge]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-ragcartridge"); var verEl = document.getElementById("ragver-" + slug); var version = (verEl && verEl.value.trim()) || "0.1.0"; ragCart[slug] = { error: "Drawing the cartridge…" }; render(); fetch("/api/rag/export-png", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ slug: slug, version: version }) }).then(function (r) { return r.json(); }).then(function (j) { ragCart[slug] = j.ok ? { file: j.file, size: j.size } : { error: j.error || "Export failed." }; render(); }).catch(function () { ragCart[slug] = { error: "The bridge didn't answer." }; render(); }); }; });
    var ragCartPick = document.getElementById("rag-cart-pick");
    if (ragCartPick) ragCartPick.onclick = function () { var f = document.getElementById("rag-cart-file"); if (f) f.click(); };
    var ragCartFile = document.getElementById("rag-cart-file");
    if (ragCartFile) ragCartFile.onchange = function () {
      var file = ragCartFile.files && ragCartFile.files[0];
      if (!file) return;
      var rd = new FileReader();
      rd.onload = function () {
        var b64 = String(rd.result || "").split(",")[1] || "";
        ragMsg = "Installing the pack from its picture — building its index here…"; render();
        fetch("/api/rag/install-cartridge", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ png: b64 }) }).then(function (r) { return r.json(); }).then(function (j) { ragMsg = j.ok ? "Installed from its picture — the pack is on this machine and active." : (j.error || "Install failed."); ragPacksState = null; render(); }).catch(function () { ragMsg = "The bridge didn't answer."; render(); });
      };
      rd.readAsDataURL(file);
    };
    mainEl.querySelectorAll("[data-ragexport]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-ragexport"); var verEl = document.getElementById("ragver-" + slug); var version = (verEl && verEl.value.trim()) || "0.1.0"; ragExp[slug] = { error: "Building the repo ZIP…" }; render(); fetch("/api/rag/export", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ slug: slug, version: version }) }).then(function (r) { return r.json(); }).then(function (j) { ragExp[slug] = j.ok ? { file: j.file, files: j.files || [] } : { error: j.error || "Export failed." }; render(); }).catch(function () { ragExp[slug] = { error: "The bridge didn't answer." }; render(); }); }; });

    var readRagFiles = function (files) {
      var list = Array.prototype.slice.call(files || []).filter(function (f) { return /.(md|txt|pdf)$/i.test(f.name) || f.type === "application/pdf" || String(f.type || "").indexOf("text/") === 0; });
      if (!list.length) { ragForgeMsg = "Those files aren't documents — the Studio takes .md, .txt and .pdf files."; render(); return; }
      var pending = list.length;
      list.forEach(function (f) {
        // A PDF travels as base64; the bridge extracts its text
        // when the pack is forged, and only the text is stored.
        var isPdf = /.pdf$/i.test(f.name) || f.type === "application/pdf";
        var rd = new FileReader();
        rd.onload = function () {
          if (isPdf) {
            var url = String(rd.result || "");
            var b64 = url.slice(url.indexOf(",") + 1);
            if (b64) {
              ragForgeDocs = ragForgeDocs.filter(function (d) { return d.title !== f.name; });
              ragForgeDocs.push({ title: f.name, pdf_base64: b64 });
            }
          } else {
            var body = String(rd.result || "");
            if (body) {
              ragForgeDocs = ragForgeDocs.filter(function (d) { return d.title !== f.name; });
              ragForgeDocs.push({ title: f.name, body: body });
            }
          }
          if (--pending === 0) render();
        };
        rd.onerror = function () { if (--pending === 0) render(); };
        if (isPdf) rd.readAsDataURL(f); else rd.readAsText(f);
      });
    };
    var rfFiles = document.getElementById("rf-files");
    if (rfFiles) {
      rfFiles.onclick = function (e) { e.stopPropagation(); };
      rfFiles.onchange = function () { readRagFiles(rfFiles.files); };
    }
    var rfDrop = document.getElementById("rf-drop");
    if (rfDrop) {
      rfDrop.onclick = function () { if (rfFiles) rfFiles.click(); };
      rfDrop.onkeydown = function (e) { if (e.key === "Enter" || e.key === " ") { e.preventDefault(); if (rfFiles) rfFiles.click(); } };
      rfDrop.ondragover = function (e) { e.preventDefault(); if (!ragDragOver) { ragDragOver = true; render(); } };
      rfDrop.ondragleave = function () { if (ragDragOver) { ragDragOver = false; render(); } };
      rfDrop.ondrop = function (e) { e.preventDefault(); ragDragOver = false; if (e.dataTransfer && e.dataTransfer.files && e.dataTransfer.files.length) readRagFiles(e.dataTransfer.files); else render(); };
    }
    var rfIconRows = function () { var el = document.getElementById("rf-icon"); return el ? el.value.split("\n").map(function (s) { return s.replace(/\s+$/, ""); }).filter(function (s) { return s.length; }) : []; };
    var rfForge = document.getElementById("rf-forge");
    if (rfForge) rfForge.onclick = function () {
      var nameEl = document.getElementById("rf-name");
      var descEl = document.getElementById("rf-desc");
      var dirEl = document.getElementById("rf-dir");
      var nm = (nameEl ? nameEl.value : "").trim();
      var ds = (descEl ? descEl.value : "").trim();
      if (ragEdit) {
        ragForgeMsg = "Saving…"; render();
        fetch("/api/rag/update-pack", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ slug: ragEdit.slug, name: nm, description: ds, documents: ragForgeDocs, icon: rfIconRows() }) })
          .then(function (r) { return r.json(); })
          .then(function (j) { if (j.ok) { ragMsg = "Saved — " + nm + " was re-indexed with the new document set" + (ragEdit && ragEdit.sourceKey ? " and is marked edited locally (it diverges from " + ragEdit.sourceKey + "). Export it to publish your version." : "."); ragEdit = null; ragEditPrefill = null; ragForgeDocs = []; ragPacksState = null; ragCatalog = null; ragTab = "packs"; } else { ragForgeMsg = j.error || "Saving failed."; } render(); })
          .catch(function () { ragForgeMsg = "The bridge didn't answer."; render(); });
        return;
      }
      ragForgeMsg = "Packaging…"; render();
      fetch("/api/rag/forge", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ name: nm, description: ds, source_dir: (dirEl ? dirEl.value : "").trim(), documents: ragForgeDocs, icon: rfIconRows() }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { ragForgeMsg = j.ok ? "Packaged — the pack is in rag-packs/, active and ready. Ask her something it knows, or list packs with /rag in chat." : (j.error || "Packaging failed."); if (j.ok) { ragForgeDocs = []; ragPacksState = null; ragTab = "packs"; } render(); })
        .catch(function () { ragForgeMsg = "The bridge didn't answer."; render(); });
    };
  } else if (view === "ChipDeck") {
    if (CONFIG.backend === "live" && !deckData) {
      fetch("/api/deck").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { deckData = j; if (view === "ChipDeck") render(); }).catch(function () {});
    }
    var deckPost = function (path, body, done) {
      fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) })
        .then(function (r) { return r.json(); })
        .then(function (j) { done(j); deckData = null; wfData = null; ragPacksState = null; if (view === "ChipDeck") render(); })
        .catch(function () { deckMsg = "The bridge could not be reached."; render(); });
    };
    var deckInstallFile = function (file) {
      if (!file) return;
      var rd = new FileReader();
      rd.onload = function () {
        var b64 = String(rd.result || "").split(",")[1] || "";
        deckMsg = "Reading the picture…"; render();
        deckPost("/api/deck/install", { png: b64 }, function (j) { deckMsg = j.ok ? "Slotted in — " + (j.name || "the picture") + (j.version ? " v" + j.version : "") + " is in the binder now." : (j.error || "That picture could not be installed."); });
      };
      rd.readAsDataURL(file);
    };
    var dz = document.getElementById("deck-drop");
    if (dz) {
      dz.addEventListener("dragover", function (e) { e.preventDefault(); dz.style.borderColor = "var(--accent)"; });
      dz.addEventListener("dragleave", function () { dz.style.borderColor = ""; });
      dz.addEventListener("drop", function (e) { e.preventDefault(); dz.style.borderColor = ""; var f = e.dataTransfer && e.dataTransfer.files && e.dataTransfer.files[0]; deckInstallFile(f); });
    }
    var dp = document.getElementById("deck-pick");
    if (dp) dp.onclick = function () { var f = document.getElementById("deck-file"); if (f) f.click(); };
    var df = document.getElementById("deck-file");
    if (df) df.onchange = function () { deckInstallFile(df.files && df.files[0]); };
    mainEl.querySelectorAll("[data-deckrun]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-deckrun"); view = "Chat"; render(); sendChat("./" + slug); }; });
    mainEl.querySelectorAll("[data-deckstudio]").forEach(function (b) { b.onclick = function () { loadChipIntoStudio(b.getAttribute("data-deckstudio"), function () { view = "Chips"; chipsTab = "studio"; deckMsg = ""; render(); }, function (msg) { deckMsg = msg; render(); }); }; });
    mainEl.querySelectorAll("[data-deckuninstall]").forEach(function (b) { b.onclick = function () { var full = b.getAttribute("data-deckuninstall"); var lc = deckData && deckData.chips ? deckData.chips.filter(function (x) { return (x.sourceKey || ("local/" + String(x.file || "").replace(/.png$/i, ""))) === full; })[0] : null; if (lc && lc.locallyEdited && !window.confirm("This chip has local edits — uninstalling removes the cartridge and your edits with it. Uninstall anyway?")) return; deckPost("/api/chips/delete", { full_name: full }, function (j) { deckMsg = j.ok ? "Uninstalled — cartridge and record are gone." : (j.error || "Uninstall failed."); }); }; });
    mainEl.querySelectorAll("[data-deckunpair]").forEach(function (b) { b.onclick = function () { deckPost("/api/mesh/peers/remove", { nodeId: b.getAttribute("data-deckunpair") }, function (j) { deckMsg = j.ok ? "Unpaired — the card picture stays in the binder; the pairing is gone." : "Unpair failed."; }); }; });
    mainEl.querySelectorAll("[data-deckragtoggle]").forEach(function (b) { b.onclick = function () { deckPost("/api/rag/toggle", { full_name: b.getAttribute("data-deckragtoggle"), active: !b.getAttribute("data-active") }, function (j) { deckMsg = j.ok ? "" : (j.error || "Toggle failed."); }); }; });
    mainEl.querySelectorAll("[data-deckragdelete]").forEach(function (b) { b.onclick = function () { deckPost("/api/rag/delete", { full_name: b.getAttribute("data-deckragdelete") }, function (j) { deckMsg = j.ok ? "Removed — the pack's documents and index are gone." : (j.error || "Remove failed."); }); }; });
  } else if (view === "Settings") {
    if (settingsTab === "system") wireUpdaterFold();
    mainEl.querySelectorAll("[data-settab]").forEach(function (b) { b.onclick = function () { settingsTab = b.getAttribute("data-settab"); render(); }; });
    if (settingsTab === "operator" || settingsTab === "persona") {
      // The two memory files' wiring (Build 6, #110; the handlers
      // are Build 4's, unchanged — same storage): Forget lives on
      // both tabs (each entry carries its true index in the one
      // list), the add row only on Operator, the leash and the
      // proposal queue only on Persona. Each handler
      // binds only where its element rendered.
      mainEl.querySelectorAll("[data-del]").forEach(function (b) { b.onclick = function () { var gone = memory[Number(b.getAttribute("data-del"))]; if (gone && gone.t) seatForgotten.push(gone.t); memory.splice(Number(b.getAttribute("data-del")), 1); persist(); render(); }; });
      var memAdd = document.getElementById("mem-add");
      if (memAdd) memAdd.onclick = function () { var inp = document.getElementById("mem-in"); var v = inp ? inp.value.trim() : ""; if (!v) return; memory.push({ t: v, o: "manual" }); persist(); render(); };
      mainEl.querySelectorAll("[data-memmode]").forEach(function (b) { b.onclick = function () { CONFIG.memoryMode = b.getAttribute("data-memmode") || "autonomous"; saveCfg(); render(); }; });
      mainEl.querySelectorAll("[data-memprop]").forEach(function (b) { b.onclick = function () { resolveProposalEverywhere(b.getAttribute("data-memprop"), true); }; });
      mainEl.querySelectorAll("[data-mempropno]").forEach(function (b) { b.onclick = function () { resolveProposalEverywhere(b.getAttribute("data-mempropno"), false); }; });
    }
    var bind = function (id, fn) { var el = document.getElementById(id); if (el) el.onclick = fn; };
    var saveVoice = function (patch) {
      if (CONFIG.backend !== "live") { voiceMsg = "Voice settings save through the live bridge — switch the backend to Live in Settings."; render(); return; }
      fetch("/api/voice/settings", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(patch) })
        .then(function (r) { return r.json(); })
        .then(function () { voiceState = null; refreshVoice(true); })
        .catch(function () { voiceMsg = "The settings couldn't be saved — the bridge didn't answer."; render(); });
    };
    var enSw = document.getElementById("v-enabled");
    if (enSw) enSw.onclick = function () { saveVoice({ enabled: !(voiceState ? voiceState.enabled : true) }); };
    var sttSel = document.getElementById("v-stt");
    if (sttSel) sttSel.onchange = function () { saveVoice({ stt_model: sttSel.value }); };
    var engSel = document.getElementById("v-engine");
    if (engSel) engSel.onchange = function () { saveVoice({ tts_engine: engSel.value, tts_voice: engSel.value === "piper" ? "en_US-lessac-medium" : "bf_isabella" }); };
    var voiceSel = document.getElementById("v-voice");
    if (voiceSel) voiceSel.onchange = function () { saveVoice({ tts_voice: voiceSel.value }); };
    var rateInp = document.getElementById("v-rate");
    if (rateInp) rateInp.onchange = function () { saveVoice({ rate: Number(rateInp.value) }); };
    var pitchInp = document.getElementById("v-pitch");
    if (pitchInp) pitchInp.onchange = function () { saveVoice({ pitch: Number(pitchInp.value) }); };
    // The call panel's wiring (Build 6, #113): bound here now
    // that the panel lives on this tab. Same ids, same handlers
    // the Voice view's branch carried; each binds only where
    // its element rendered, so they ride along harmlessly on
    // the tab's other renders. (The retired pointer card's
    // #voice-goto-settings binding went with the card.)
    var startBtn = document.getElementById("call-start");
    if (startBtn) startBtn.onclick = function () { startCall(); };
    var endBtn = document.getElementById("call-end");
    if (endBtn) endBtn.onclick = function () { endCall(); };
    var pttBtn = document.getElementById("call-ptt");
    if (pttBtn) pttBtn.onclick = function () { pttToggle("call"); };
    var sayBtn = document.getElementById("call-say");
    if (sayBtn) sayBtn.onclick = function () { var inp = document.getElementById("call-type"); var t = inp ? inp.value.trim() : ""; if (t) { if (inp) inp.value = ""; callNaviTurn(t); } };
    var typeInp = document.getElementById("call-type");
    if (typeInp) typeInp.onkeydown = function (e) { if (e.key === "Enter") { e.preventDefault(); var b = document.getElementById("call-say"); if (b) b.click(); } };
    var testBtn = document.getElementById("voice-test");
    if (testBtn) testBtn.onclick = function () {
      testBtn.textContent = "Synthesizing…"; testBtn.disabled = true;
      if (CONFIG.backend !== "live") { mockSpeak("Hey — it's me, Navi. This is the mock stand-in voice; the real one runs locally through the live bridge.").then(function () { render(); }); return; }
      fetch("/api/voice/test", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j.ok && j.audio_b64) return playWavB64(j.audio_b64); voiceMsg = j.error || "She couldn't speak just then."; })
        .catch(function () { voiceMsg = "The bridge didn't answer the voice test."; })
        .then(function () { render(); });
    };
    var speakRep = document.getElementById("v-speakreplies");
    if (speakRep) speakRep.onchange = function () { CONFIG.speakReplies = speakRep.checked; saveCfg(); };
    bind("gh-signin", ghSignInStart);
    bind("gh-complete", ghSignInCompleteMock);
    bind("gh-cancel", function () { ghStopPolling(); githubDevice = null; ghSetMsg("", false); render(); });
    bind("gh-signout", ghSignOut);
    bind("gh-settings-save", ghSaveBackupSettings);
    bind("gh-backup", ghBackupNow);
    var scSw = document.getElementById("gh-sync-choice");
    if (scSw) scSw.onclick = function () { syncChoiceFlip(); };
    bind("gh-preview", ghImportPreview);
    bind("gh-apply", ghImportApply);
    bind("gh-badge-sync", ghBadgeSync);
    // Build 13 (#136/#150): the Account sync card's own badge
    // slot — the shared intake (badgeSlotFile), the same single
    // badge state the Operator room's slot feeds.
    var ghSlotBtn = document.getElementById("gh-badge-slot");
    var ghSlotFile = document.getElementById("gh-badge-file");
    if (ghSlotBtn && ghSlotFile) {
      ghSlotBtn.onclick = function () { ghSlotFile.click(); };
      ghSlotFile.addEventListener("change", function () {
        var f = ghSlotFile.files && ghSlotFile.files[0];
        ghSlotFile.value = "";
        if (!f) return;
        badgeSlotFile(f, function (okSlot, msg) { ghSetMsg(msg, !okSlot); render(); });
      });
    }
    var visSel = document.getElementById("gh-vis");
    var paintPubWarn = function () {
      var w = document.getElementById("gh-pubwarn");
      if (w) w.innerHTML = visSel && visSel.value === "public" ? "<div class='card' style='border-color:#f87171;color:#f87171;margin-top:8px'>A public backup repo puts her Soul and Memory documents on the open internet for anyone to read. Only choose this if you truly mean to publish them.</div>" : "";
    };
    if (visSel) { visSel.onchange = paintPubWarn; paintPubWarn(); }
    var th = document.getElementById("th");
    if (th) th.onchange = function () { CONFIG.theme = th.value; saveCfg(); applyTheme(); render(); };
    var lay = document.getElementById("lay");
    if (lay) lay.onchange = function () { setLayout(lay.value); };
    var mp = document.getElementById("motion-potato");
    if (mp) mp.onclick = function () { CONFIG.motion = CONFIG.motion === "potato" ? "full" : "potato"; saveCfg(); applyMotion(); render(); };
    mainEl.querySelectorAll("[data-uiscale]").forEach(function (b) { b.onclick = function () { CONFIG.uiScale = Number(b.getAttribute("data-uiscale")) || 1; saveCfg(); applyUiScale(); render(); }; });
    var opSave = document.getElementById("op-save");
    if (opSave) opSave.onclick = function () { var inp = document.getElementById("op-name"); CONFIG.operatorName = inp ? inp.value.trim() : ""; saveCfg(); refreshSeedGreeting(); render(); };
    // Issue #116: the operator's picture and Operator Badge.
    var opPicPick = document.getElementById("op-pic-pick");
    var opPicFile = document.getElementById("op-pic-file");
    if (opPicPick && opPicFile) opPicPick.onclick = function () { opPicFile.click(); };
    if (opPicFile) opPicFile.onchange = function () {
      var f = opPicFile.files && opPicFile.files[0];
      if (!f) return;
      var rd = new FileReader();
      rd.onload = function () {
        var img = new Image();
        img.onload = function () {
          try {
            var cv = document.createElement("canvas"); cv.width = 256; cv.height = 256;
            var cx = cv.getContext("2d");
            var s = Math.max(256 / img.width, 256 / img.height);
            cx.drawImage(img, (256 - img.width * s) / 2, (256 - img.height * s) / 2, img.width * s, img.height * s);
            CONFIG.operatorPicture = cv.toDataURL("image/jpeg", 0.85);
          } catch (e) { CONFIG.operatorPicture = String(rd.result || ""); }
          saveCfg(); render();
        };
        img.onerror = function () { badgeMsg = "That file would not open as a picture."; render(); };
        img.src = String(rd.result || "");
      };
      rd.readAsDataURL(f);
    };
    var opPicClear = document.getElementById("op-pic-clear");
    if (opPicClear) opPicClear.onclick = function () { CONFIG.operatorPicture = ""; saveCfg(); render(); };
    var opFacePhoto = document.getElementById("op-badge-face-photo");
    if (opFacePhoto) opFacePhoto.onclick = function () { badgeFaceChoice = "photo"; render(); };
    var opFaceArt = document.getElementById("op-badge-face-art");
    if (opFaceArt) opFaceArt.onclick = function () { badgeFaceChoice = "art"; render(); };
    var opBadgeMint = document.getElementById("op-badge-mint");
    if (opBadgeMint) opBadgeMint.onclick = function () {
      refreshBadgeInfo();
      badgeMsg = "Minting your badge…"; render();
      var finishMint = function (faceImg) {
        badgeForge(faceImg, opName(), badgeNodeName, function (url) {
          if (url) { CONFIG.operatorBadge = url; badgeMsg = "Badge minted — it rides in your settings backup, and chat can wear its face."; }
          else badgeMsg = "The badge could not be minted in this browser.";
          saveCfg(); render();
        });
      };
      if (badgeFaceChoice === "photo" && CONFIG.operatorPicture) {
        var pim = new Image();
        pim.onload = function () { finishMint(pim); };
        pim.onerror = function () { finishMint(null); };
        pim.src = CONFIG.operatorPicture;
      } else finishMint(null);
    };
    var opBadgeExport = document.getElementById("op-badge-export");
    if (opBadgeExport) opBadgeExport.onclick = function () {
      if (!CONFIG.operatorBadge) return;
      var a = document.createElement("a");
      a.href = CONFIG.operatorBadge; a.download = "operator-badge.png"; a.click();
    };
    var opBadgeClear = document.getElementById("op-badge-clear");
    if (opBadgeClear) opBadgeClear.onclick = function () { CONFIG.operatorBadge = ""; badgeMsg = ""; saveCfg(); render(); };
    // Build 13 (#150): the Operator room's badge slot — a badge
    // minted on another machine slots here from its picture
    // file alone, into the same single badge state.
    refreshBadgeInfo();
    var opSlotBtn = document.getElementById("op-badge-slot");
    var opSlotFile = document.getElementById("op-badge-file");
    if (opSlotBtn && opSlotFile) {
      opSlotBtn.onclick = function () { opSlotFile.click(); };
      opSlotFile.addEventListener("change", function () {
        var f = opSlotFile.files && opSlotFile.files[0];
        opSlotFile.value = "";
        if (!f) return;
        badgeSlotFile(f, function (okSlot, msg) { badgeMsg = msg; render(); });
      });
    }
    // Hardware profile card: load the profile on first Settings open.
    if (!hardwareState) fetchHardware(true);
    // Prefill override inputs from the profile.
    var hwProf = hardwareState && hardwareState.profile ? hardwareState.profile : null;
    if (hwProf && hwProf.settings) {
      var hwSet = function (id, v) { var el = document.getElementById(id); if (el && v != null) el.value = v; };
      hwSet("hw-thr", hwProf.settings.threads); hwSet("hw-ctx", hwProf.settings.ctxSize); hwSet("hw-bat", hwProf.settings.batchSize); hwSet("hw-gpu", hwProf.settings.gpuLayers);
    }
    var hwSave = document.getElementById("hw-save");
    if (hwSave) hwSave.onclick = function () {
      if (CONFIG.backend !== "live") { hardwareMsg = "Overrides need the live bridge — switch Backend to Live."; render(); return; }
      var num = function (id) { var el = document.getElementById(id); return el ? parseInt(el.value, 10) : NaN; };
      fetch("/api/hardware/override", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ threads: num("hw-thr"), ctxSize: num("hw-ctx"), batchSize: num("hw-bat"), gpuLayers: num("hw-gpu") }) }).then(function (r) { return r.json(); }).then(function (j) { hardwareState = j; hardwareMsg = j && j.ok === false ? ("Override refused: " + (j.error || "invalid values")) : "Override saved — it applies the next time the model server starts."; render(); }).catch(function () { hardwareMsg = "The bridge did not answer — is it running?"; render(); });
    };
    var hwRedetect = document.getElementById("hw-redetect");
    if (hwRedetect) hwRedetect.onclick = function () {
      if (CONFIG.backend !== "live") { hardwareMsg = "Re-detect needs the live bridge — switch Backend to Live."; render(); return; }
      fetch("/api/hardware/redetect", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).then(function (r) { return r.json(); }).then(function (j) { hardwareState = j; hardwareMsg = j && j.profile ? "Re-detected this machine and returned to auto settings — they apply the next time the model server starts." : "Re-detect failed: " + ((j && j.error) || "unknown error"); render(); }).catch(function () { hardwareMsg = "The bridge did not answer — is it running?"; render(); });
    };
    wireApexNode();
    wireMeshCard();
    var bootSettingsBtn = document.getElementById("boot-action-settings");
    if (bootSettingsBtn) bootSettingsBtn.onclick = function () { bootAction(); };
    if (settingsTab === "system" && !bootStateCache && !bootFetching && CONFIG.backend === "live") {
      bootFetching = true;
      fetch("/api/status").then(function (r) { return r.json(); }).then(function (j) { bootFetching = false; if (j && j.boot) { bootStateCache = j.boot; if (view === "Settings" && settingsTab === "system") render(); } }).catch(function () { bootFetching = false; });
    }
    wireDoorDevices();
    wireSudoCard();
    var savePersona = function () { saveCfg(); if (CONFIG.backend === "live") fetch("/api/persona", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ soul: CONFIG.soul, memory: CONFIG.memory }) }).catch(function () {}); };
    var soulSave = document.getElementById("soul-save");
    if (soulSave) soulSave.onclick = function () { var ta = document.getElementById("soul-doc"); if (ta) CONFIG.soul = ta.value; savePersona(); };
    var memSave = document.getElementById("memdoc-save");
    if (memSave) memSave.onclick = function () { var ta = document.getElementById("mem-doc"); if (ta) CONFIG.memory = ta.value; savePersona(); };
    mainEl.querySelectorAll("[data-inst]").forEach(function (b) { b.onclick = function () { var n = b.getAttribute("data-inst"); if ((CONFIG.installed || []).indexOf(n) < 0) { CONFIG.installed = (CONFIG.installed || []).concat([n]); saveCfg(); } render(); }; });
    mainEl.querySelectorAll("[data-uninst]").forEach(function (b) { b.onclick = function () { var n = b.getAttribute("data-uninst"); CONFIG.installed = (CONFIG.installed || []).filter(function (x) { return x !== n; }); saveCfg(); render(); }; });
    var cmdAdd = document.getElementById("cmd-add");
    if (cmdAdd) cmdAdd.onclick = function () { var inp = document.getElementById("cmd-custom"); var n = inp ? inp.value.trim().toLowerCase().replace(/[^a-z0-9-]/g, "") : ""; if (n && (CONFIG.installed || []).indexOf(n) < 0) { CONFIG.installed = (CONFIG.installed || []).concat([n]); saveCfg(); } render(); };
  } else if (view === "Guardian") {
    // Guardian wiring (Build 14, #156): the Save button posts
    // the full settings object and the view re-renders from
    // the state the bridge returns; the poll starts here, in
    // the view's own wiring, and render() stops it the moment
    // the view changes.
    var guardianSaveBtn = document.getElementById("guardian-save");
    if (guardianSaveBtn) guardianSaveBtn.onclick = function () { guardianSaveSettings(); };
    guardianStartPolling();
  } else if (view === "GitHub") {
    var bind = function (id, fn) { var el = document.getElementById(id); if (el) el.onclick = fn; };
    bind("gh-tab-overview", function () { githubTab = "overview"; if (!githubPulse && !githubPulseFetching) fetchGithubPulse(false); render(); });
    bind("gh-tab-issues", function () { githubTab = "issues"; if (!githubIssues && !githubIssuesFetching) fetchGithubIssues(false); render(); });
    bind("gh-tab-roadmap", function () { githubTab = "roadmap"; openGithubPageWindow("roadmap"); render(); });
    bind("gh-tab-discussions", function () { githubTab = "discussions"; openGithubPageWindow("discussions"); render(); });
    bind("gh-webview-open", function () { openGithubPageWindow(githubTab); });
    mainEl.querySelectorAll("[data-gh-open]").forEach(function (b) { b.onclick = function () { var u = b.getAttribute("data-gh-open"); if (u && (u.indexOf("https://github.com/") === 0 || u.indexOf("https://github.com/users/") === 0)) openExternal(u); }; });
    bind("gh-pulse-refresh", function () { githubPulseError = ""; fetchGithubPulse(true); });
    bind("gh-issues-refresh", function () { githubIssuesError = ""; fetchGithubIssues(true); });
    bind("gh-repo", function () { openExternal(NAVI_REPO_URL); });
    bind("gh-issues-page", function () { openExternal(NAVI_REPO_URL + "/issues"); });
    bind("gh-legacy", function () { openExternal("https://github.com/SaintsSec/Navi"); });
    mainEl.querySelectorAll("[data-gh-issue]").forEach(function (b) { b.onclick = function () { var u = b.getAttribute("data-gh-issue"); if (!u || u.indexOf("https://github.com/") !== 0) return; var m = /^https:\/\/github\.com\/AlexKollar\/Apex\/issues\/(\d+)$/.exec(u); if (m && docsInShell()) { openGithubIssueWindow(m[1], u); return; } openExternal(u); }; });
    if (githubTab === "overview" && !githubPulse && !githubPulseFetching && Date.now() - githubPulseFailedAt > 30000) fetchGithubPulse(true);
    if (githubTab === "issues" && !githubIssues && !githubIssuesFetching && Date.now() - githubIssuesFailedAt > 30000) fetchGithubIssues(true);
  }
  if (view === "Chips" || view === "RAG") {
    // Publish to GitHub + README editing (packs and chips, one loop).
    function pubStore(kind) { return kind === "rag" ? ragPub : chipPub; }
    function readmeStore(kind) { return kind === "rag" ? ragReadmeEd : chipReadmeEd; }
    function pubVersion(kind, key) {
      var id = kind === "rag" ? "ragver-" + key : "chipver-" + String(key).replace(/\.png$/i, "");
      var el = document.getElementById(id);
      return (el && el.value.trim()) || "0.1.0";
    }
    mainEl.querySelectorAll("[data-ragpub]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-ragpub"); var p = (ragPacksState || []).find(function (x) { return x.slug === slug; }); ragPub[slug] = ragPub[slug] && ragPub[slug].open ? { open: false } : { open: true, repo: repoSlug(p ? p.name : slug) }; render(); }; });
    mainEl.querySelectorAll("[data-chippub]").forEach(function (b) { b.onclick = function () { var file = b.getAttribute("data-chippub"); var c = wfData && wfData.chips ? wfData.chips.find(function (x) { return x.file === file; }) : null; chipPub[file] = chipPub[file] && chipPub[file].open ? { open: false } : { open: true, repo: repoSlug(c ? c.name : String(file).replace(/\.png$/i, "")) }; render(); }; });
    mainEl.querySelectorAll("[data-pubrepo]").forEach(function (inp) { inp.oninput = function () { var parts = inp.getAttribute("data-pubrepo").split(":"); var st = pubStore(parts[0])[parts.slice(1).join(":")]; if (st) st.repo = inp.value; }; });
    mainEl.querySelectorAll("[data-pubgo]").forEach(function (b) { b.onclick = function () { var parts = b.getAttribute("data-pubgo").split(":"); var kind = parts[0]; var key = parts.slice(1).join(":"); var st = pubStore(kind)[key]; if (!st) return; st.busy = true; st.msg = ""; st.err = ""; render(); var url = kind === "rag" ? "/api/rag/publish" : "/api/chips/publish"; var body = kind === "rag" ? { slug: key, version: pubVersion(kind, key), repo_name: st.repo } : { file: key, version: pubVersion(kind, key), repo_name: st.repo }; fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) { st.busy = false; if (j.ok) { st.msg = (j.created ? "Repo created: " : "Repo updated: ") + (j.repo || st.repo) + (j.release ? " — release " + j.release.tag + " is published" + (j.release.asset ? ", with " + j.release.asset + " attached" : "") + "." : " — files uploaded.") + (j.assetSkipped ? " " + j.assetSkipped : ""); st.url = (j.release && j.release.url) || j.repoUrl || ""; st.err = ""; } else { st.err = j.error || "Publish failed."; } render(); }).catch(function () { st.busy = false; st.err = "The bridge didn't answer."; render(); }); }; });
    function openReadme(kind, key) { var st = readmeStore(kind)[key]; if (st && st.open) { readmeStore(kind)[key] = { open: false }; render(); return; } readmeStore(kind)[key] = { open: true, loading: true, preview: true, text: "", edited: false, msg: "", err: "" }; render(); var url = kind === "rag" ? "/api/rag/readme" : "/api/chips/readme"; var body = kind === "rag" ? { slug: key, version: pubVersion(kind, key) } : { file: key, version: pubVersion(kind, key) }; fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) { var cur = readmeStore(kind)[key]; if (!cur || !cur.open) return; cur.loading = false; if (j.ok) { cur.text = j.readme || ""; cur.edited = !!j.edited; } else { cur.err = j.error || "The README could not be loaded."; } render(); }).catch(function () { var cur = readmeStore(kind)[key]; if (cur) { cur.loading = false; cur.err = "The bridge didn't answer."; render(); } }); }
    mainEl.querySelectorAll("[data-ragreadme]").forEach(function (b) { b.onclick = function () { openReadme("rag", b.getAttribute("data-ragreadme")); }; });
    mainEl.querySelectorAll("[data-chipreadme]").forEach(function (b) { b.onclick = function () { openReadme("chip", b.getAttribute("data-chipreadme")); }; });
    mainEl.querySelectorAll("[data-readmearea]").forEach(function (ta) { var pre = ta.parentElement ? ta.parentElement.querySelector("pre") : null; ta.oninput = function () { var parts = ta.getAttribute("data-readmearea").split(":"); var st = readmeStore(parts[0])[parts.slice(1).join(":")]; if (st) st.text = ta.value; if (pre) { pre.innerHTML = highlightMd(ta.value) + "\n"; pre.scrollTop = ta.scrollTop; pre.scrollLeft = ta.scrollLeft; } }; ta.onscroll = function () { if (pre) { pre.scrollTop = ta.scrollTop; pre.scrollLeft = ta.scrollLeft; } }; });
    // Press-inside opens the README editor (v0.5.11e,
    // #284): the README opens rendered, like every MD
    // document in Apex now; a press in its text swaps to
    // the editor, and Save returns it to the viewer.
    // Presses on links and buttons inside keep their acts.
    mainEl.querySelectorAll("[data-readmepress]").forEach(function (pv) { pv.onclick = function (ev) { var t = ev.target; if (t.closest && t.closest("a, button, input, select, textarea, label")) return; var parts = pv.getAttribute("data-readmepress").split(":"); var st = readmeStore(parts[0])[parts.slice(1).join(":")]; if (st) { st.preview = false; render(); mainEl.querySelectorAll("[data-readmearea]").forEach(function (ta) { if (ta.getAttribute("data-readmearea") === pv.getAttribute("data-readmepress")) ta.focus(); }); } }; });
    mainEl.querySelectorAll("[data-readmesave]").forEach(function (b) { b.onclick = function () { var parts = b.getAttribute("data-readmesave").split(":"); var kind = parts[0]; var key = parts.slice(1).join(":"); var st = readmeStore(kind)[key]; if (!st) return; var url = kind === "rag" ? "/api/rag/readme-save" : "/api/chips/readme-save"; var body = kind === "rag" ? { slug: key, readme: st.text } : { file: key, readme: st.text }; fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) { if (j.ok) { st.edited = true; st.preview = true; st.msg = "Saved — this README now ships with it."; st.err = ""; } else { st.err = j.error || "Save failed."; } render(); }).catch(function () { st.err = "The bridge didn't answer."; render(); }); }; });
    mainEl.querySelectorAll("[data-readmerestore]").forEach(function (b) { b.onclick = function () { var parts = b.getAttribute("data-readmerestore").split(":"); var kind = parts[0]; var key = parts.slice(1).join(":"); var url = kind === "rag" ? "/api/rag/readme-restore" : "/api/chips/readme-restore"; var body = kind === "rag" ? { slug: key } : { file: key }; fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function () { readmeStore(kind)[key] = { open: false }; openReadme(kind, key); }).catch(function () { render(); }); }; });
    mainEl.querySelectorAll("[data-readmeclose]").forEach(function (b) { b.onclick = function () { var parts = b.getAttribute("data-readmeclose").split(":"); readmeStore(parts[0])[parts.slice(1).join(":")] = { open: false }; render(); }; });
  }
}
// Link routing (build .48, issue #10): plain anchors did nothing
// inside the standalone shell — WebKitGTK drops target=_blank and
// plain navigation out of the webview without a word, so every link
// on the page (card links, the Updater's release-notes link, the
// GitHub device sign-in link, links inside her markdown replies)
// died on the click while the same page in a browser opened them
// all. One delegated handler, installed once here at startup,
// routes each anchor click by where its href points. Left alone,
// with their default behavior standing: non-left-button clicks,
// clicks with ctrl/meta/shift/alt held, empty or javascript:
// hrefs, pure in-page #fragment hrefs, and same-origin addresses
// that are not the docs pop-out form. Download anchors keep their
// native behavior in a browser — but inside the shell the webview
// cannot save a file at all, so there a download click (a download
// attribute, a blob: address, or one of the bridge's file/export
// addresses) runs the save flow below instead: the dashboard
// fetches the bytes itself and the bridge writes them into the
// operator's Downloads folder, saying where they landed.
function naviFileEndpoint(href) {
  try {
    var u = new URL(href, location.href);
    if (u.origin !== location.origin) return false;
    return u.pathname === "/api/chips/file" || u.pathname === "/api/chips/export-file" || u.pathname === "/api/rag/export-file";
  } catch (e) { return false; }
}
function naviDownloadName(href, attrName) {
  if (attrName) return attrName;
  try {
    var u = new URL(href, location.href);
    var n = u.searchParams.get("name");
    if (n) return n;
    var seg = u.pathname.split("/").filter(function (s) { return !!s; });
    if (seg.length) return decodeURIComponent(seg[seg.length - 1]);
  } catch (e) {}
  return "navi-download";
}
function naviB64(buf) {
  var bytes = new Uint8Array(buf);
  var out = "";
  var CHUNK = 32768;
  for (var i = 0; i < bytes.length; i += CHUNK) {
    out += String.fromCharCode.apply(null, bytes.subarray(i, Math.min(i + CHUNK, bytes.length)));
  }
  return btoa(out);
}
function naviDownloadNote(text) {
  if (view === "RAG") ragMsg = text; else chipMsg = text;
  render();
}
function saveShellDownload(href, attrName) {
  var name = naviDownloadName(href, attrName);
  naviDownloadNote("Saving " + name + " to the Downloads folder…");
  // Issue #10's standing rule covers downloads too: a failed
  // save in the desktop app ends at the warning toast — the
  // target address plus the Web UI stand-in — alongside the
  // in-view note that names the reason.
  var absHref = href;
  try { absHref = new URL(href, location.href).href; } catch (e) {}
  var fail = function (reason) {
    naviDownloadNote("That download could not be saved: " + reason);
    naviToast("That download could not be opened — the desktop app could not save it (" + reason + "). The address is: " + absHref + " Stand-in until this is fixed: open the Apex Web UI (http://localhost:8675) in your browser — links work there.");
  };
  fetch(href).then(function (r) {
    if (!r.ok) throw new Error("the bridge answered " + r.status);
    return r.arrayBuffer();
  }).then(function (buf) {
    return fetch("/api/downloads/save", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ filename: name, dataBase64: naviB64(buf) }) });
  }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.ok) naviDownloadNote("Saved " + name + " — " + (j.path || "in the Downloads folder") + ".");
    else fail((j && j.error) || "the bridge did not answer.");
  }).catch(function (err) {
    fail(err && err.message ? err.message : "the fetch failed.");
  });
}
document.addEventListener("click", function (e) {
  if (e.button !== 0 || e.ctrlKey || e.metaKey || e.shiftKey || e.altKey) return;
  var el = e.target;
  while (el && el.nodeType === 1 && String(el.tagName).toUpperCase() !== "A") el = el.parentNode;
  if (!el || el.nodeType !== 1 || String(el.tagName).toUpperCase() !== "A") return;
  var href = el.getAttribute("href") || "";
  if (docsInShell() && href && (el.hasAttribute("download") || href.toLowerCase().indexOf("blob:") === 0 || naviFileEndpoint(href))) {
    e.preventDefault();
    saveShellDownload(href, el.getAttribute("download") || "");
    return;
  }
  if (el.hasAttribute("download")) return;
  if (!href || href.toLowerCase().indexOf("javascript:") === 0) return;
  var url;
  try { url = new URL(href, location.href); } catch (err) { return; }
  if (url.protocol !== "http:" && url.protocol !== "https:") return;
  var popdocs = url.hash.indexOf("popdocs") !== -1 || url.search.indexOf("popdocs") !== -1;
  if (url.origin === location.origin) {
    if (!popdocs) return;
    // The docs pop-out form: inside the shell, run the same
    // open_docs_window flow the Docs view's own pop-out button
    // uses; in a browser, open the address in its own window.
    e.preventDefault();
    if (docsInShell()) {
      if (docsShellOut) { docsTauriInvoke("open_docs_window").catch(function () {}); return; }
      docsShellOut = true;
      docsTauriInvoke("open_docs_window").then(function () { docsShellOut = true; }).catch(function () { docsShellOut = false; docsPopBlocked = true; render(); naviToast("The documentation window could not be opened — the shell did not answer. Use the direct link in the Docs view instead. The address is: " + url.href + " Stand-in until this is fixed: open the Apex Web UI (http://localhost:8675) in your browser — links work there."); });
    } else {
      window.open(url.href, "_blank");
    }
    return;
  }
  // Another site entirely: the app itself never navigates away.
  // Inside the shell the link opens in the real browser through
  // openExternal; in a browser an anchor already aimed at a new
  // tab keeps its default, and any other anchor gets a new tab
  // rather than replacing the dashboard.
  if (docsInShell()) { e.preventDefault(); openExternal(url.href); return; }
  if (el.getAttribute("target") === "_blank") return;
  e.preventDefault();
  window.open(url.href, "_blank", "noopener");
});
// In live mode, pick up persona edits made directly to the config
// files since the app last ran — and start the chip catalog search
// in the background, so the Navi User Repo list has auto-populated by
// the time the Chips screen is opened.
if (CONFIG.backend === "live") {
  loadChipCatalog();
  hydrateChatsFromServer();
  noteSeat();
  // One real update check per session, shortly after load, so the
  // status column's UPDATER element can light up on its own when a
  // newer release exists — the operator never has to go looking.
  checkForUpdates();
  // And pick up the last apply's record: if this page load is the
  // relaunched app after a one-click update, the panel's first
  // paint already carries its "Updated vX → vY" confirmation (or
  // the failure and its fallback).
  fetchUpdateResult(false);
  fetch("/api/persona").then(function (r) { return r.json(); }).then(function (j) {
    if (j.soul) CONFIG.soul = j.soul;
    if (j.memory) CONFIG.memory = j.memory;
  }).catch(function () {}).then(function () { render(); });
} else {
  render();
}
</script>
</body>
</html>
