# app/index.html — part 2 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 2422–4954 of app/index.html.

// code editor), and Workflows (chips chained output → input).
var chipsTab = "management";
var chipCatalog = null;      // last /api/chips/catalog answer
var chipCatalogBusy = false;
// NUR topic tabs: SaintsSec's own chips vs everyone else's, split
// by repo owner. The Community tab's search text filters its list
// only while something is typed; empty means the full list.
var nurTab = "saintssec";
var nurQuery = "";
// The catalog auto-populates: this runs once at app launch (live
// mode) so the Navi User Repo list is already there when the Chips
// screen opens, again as a backstop when the screen opens empty,
// and from the Search NUR button as a manual refresh.
function loadChipCatalog() {
  if (CONFIG.backend !== "live" || chipCatalogBusy) return;
  chipCatalogBusy = true;
  if (view === "Chips") render();
  fetch("/api/chips/catalog").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { chipCatalog = j; chipCatalogBusy = false; if (view === "Chips") render(); }).catch(function (e) { chipCatalog = { error: "GitHub search failed: " + e.message, fetchedAt: null, chips: [] }; chipCatalogBusy = false; if (view === "Chips") render(); });
}
var chipMsg = "";
// RAG screen: two tabs — Packs (the GitHub navi-rag catalog +
// installs + active toggles, served live by the bridge) and RAG
// Studio (package documents from disk into a knowledge pack).
var ragTab = "packs";
var ragCatalog = null;       // last /api/rag/catalog answer
var ragCatalogBusy = false;
var ragMsg = "";
var ragPacksState = null;    // last /api/rag/packs answer's packs
var ragPacksFetching = false;
var ragForgeDocs = [];       // documents picked in RAG Studio [{title, body}]
var ragDragOver = false;     // RAG Studio drop zone highlight while dragging
var ragForgeMsg = "";
var ragExp = {};             // GitHub export results by pack slug {file, files} or {error}
var ragCart = {};            // cartridge export results by pack slug {file, size} or {error}
var deckData = null;         // live: last /api/deck answer (ChipDeck)
var deckMsg = "";
var ragPub = {};             // pack slug -> Publish to GitHub panel state
var ragReadmeEd = {};        // pack slug -> README editor state
var chipPub = {};            // chip file -> Publish to GitHub panel state
var chipReadmeEd = {};       // chip file -> README editor state
function repoSlug(n) { return String(n || "").toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "") || "repo"; }
function pubPanelHtml(pub, kind, key) {
  if (!pub || !pub.open) return "";
  var h = "<div style='margin-top:8px;border-top:1px solid var(--border);padding-top:8px'>";
  if (!githubState || !githubState.signedIn) {
    h += "<span style='font-size:12px'>Publish to GitHub needs a GitHub sign-in — sign in under <b>Settings → GitHub &amp; Backup</b>, or use <b>Export for GitHub</b> and upload the ZIP yourself.</span>";
  } else {
    h += "<label style='font-size:12px'>Repo name <input class='mono' data-pubrepo='" + kind + ":" + esc(key) + "' value='" + esc(pub.repo || "") + "' style='width:220px' aria-label='Repository name'></label> <button class='act' data-pubgo='" + kind + ":" + esc(key) + "'" + (pub.busy ? " disabled" : "") + ">" + (pub.busy ? "Publishing…" : "Publish") + "</button>";
    h += "<br><span style='color:var(--dim);font-size:12px'>Navi does the whole chain: creates the repo if it is new (public — the catalog only finds public repos), sets the topic automatically, uploads the same files the export builds, then creates the tag and publishes the release with the cartridge picture attached. Nothing is left to do on GitHub.</span>";
  }
  if (pub.msg) h += "<br><span style='font-size:12px;color:#34d399'>" + esc(pub.msg) + (pub.url ? " <a class='ghost' href='" + esc(pub.url) + "' target='_blank' rel='noreferrer'>Open the repo</a>" : "") + "</span>";
  if (pub.err) h += "<br><span style='font-size:12px;color:#f5b544'>" + esc(pub.err) + "</span>";
  return h + "</div>";
}
function readmeEdHtml(ed, kind, key) {
  if (!ed || !ed.open) return "";
  var h = "<div style='margin-top:8px;border-top:1px solid var(--border);padding-top:8px'><b style='font-size:12px'>README.md</b> <span style='color:var(--dim);font-size:12px'>" + (ed.edited ? "your edited version — it ships in the export and the upload alike" : "the generated version") + "</span>";
  if (ed.loading) h += "<br><span style='color:var(--dim);font-size:12px'>Loading the README…</span>";
  else if (ed.preview) h += "<div class='md flip-scroll' data-readmepress='" + kind + ":" + esc(key) + "' title='Press inside the text to edit' style='border:1px solid var(--line);border-radius:6px;padding:10px 12px;max-height:280px;overflow:auto;margin-top:6px;cursor:text'>" + mdToHtml(ed.text || "") + "</div>";
  else h += "<div class='mded' style='margin-top:6px'><pre aria-hidden='true'>" + highlightMd(ed.text || "") + "\n</pre><textarea data-readmearea='" + kind + ":" + esc(key) + "' spellcheck='false' aria-label='README markdown editor'>" + esc(ed.text || "") + "</textarea></div>";
  h += "<div style='margin-top:6px'><button class='act' data-readmesave='" + kind + ":" + esc(key) + "'>Save README</button> <button class='ghost' data-readmerestore='" + kind + ":" + esc(key) + "'>Restore generated README</button> <button class='ghost' data-readmeclose='" + kind + ":" + esc(key) + "'>Close</button>" + (ed.msg ? " <span style='font-size:12px;color:#34d399'>" + esc(ed.msg) + "</span>" : "") + (ed.err ? " <span style='font-size:12px;color:#f5b544'>" + esc(ed.err) + "</span>" : "") + "</div></div>";
  return h;
}
var ragEdit = null;          // {slug, sourceKey} while RAG Studio edits an existing pack
var ragEditPrefill = null;   // {name, description, icon} consumed by the Studio form in edit mode
// The packs catalog auto-populates like the chips one: when the
// RAG screen opens empty, and from its Search GitHub button.
function loadRagCatalog() {
  if (CONFIG.backend !== "live" || ragCatalogBusy) return;
  ragCatalogBusy = true;
  if (view === "RAG") render();
  fetch("/api/rag/catalog").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { ragCatalog = j; ragCatalogBusy = false; if (view === "RAG") render(); }).catch(function (e) { ragCatalog = { error: "GitHub search failed: " + e.message, fetchedAt: null, packs: [] }; ragCatalogBusy = false; if (view === "RAG") render(); });
}
function refreshRagPacks(thenRender) {
  if (CONFIG.backend !== "live" || ragPacksFetching) return;
  ragPacksFetching = true;
  fetch("/api/rag/packs").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) {
      ragPacksState = (j && j.packs) || []; ragPacksFetching = false; if (thenRender) render();
    })
    .catch(function () { ragPacksFetching = false; });
}
var studioMade = null;       // {filename} after a successful forge
// GitHub view state (build .35): the last /api/github answer in live
// mode; in mock mode a simulated copy kept in localStorage, labeled
// simulated wherever it acts. Sign-in is optional and feature-scoped
// (Device Flow) — it gates nothing but the GitHub-backed actions.
var NAVI_REPO_URL = "https://github.com/AlexKollar/Apex";
// Alex's project board — the GitHub section's Roadmap tab. A user
// project (not a repo page), so it lives outside NAVI_REPO_URL.
var NAVI_ROADMAP_URL = "https://github.com/users/AlexKollar/projects/9";
var GH_ITEMS = [
  { id: "soul", label: "Soul document", blurb: "Who she is — her persona.", core: true },
  { id: "memory", label: "Memory document", blurb: "What she carries between sessions.", core: true },
  { id: "settings", label: "Settings & config", blurb: "Operator name, theme, installed commands, voice choice and voice settings.", core: true },
  { id: "chips", label: "Installed chips", blurb: "The full chips — restored working, not just named." },
  { id: "rag", label: "RAG packs", blurb: "Their documents, built indexes, and which packs are active." },
  { id: "workflows", label: "Workflows", blurb: "Her saved chains of chips." },
  { id: "vault", label: "The Vault", blurb: "The shared desk of notes — hers and yours, as plain markdown files." },
  { id: "mesh-cards", label: "Mesh pairing cards", blurb: "The pairing-card pictures — hers, minted fresh, and one for every paired machine. Her node keypair never rides." },
  { id: "chats", label: "Chats", blurb: "Your conversations with her — they live on the server now, and ride along so another machine picks up mid-thread. A restore converges them: newest wins per thread, and a diverged thread is kept whole, never merged away." }
];
// Soul, Memory, and Settings & config are ticked automatically
// (build .36): every backup carries them, so any item list —
// stored, loaded, or about to be saved — is folded through this.
var GH_CORE_ITEMS = ["soul", "memory", "settings"];
function ghWithCore(items) {
  return GH_ITEMS.map(function (i) { return i.id; }).filter(function (id) { return GH_CORE_ITEMS.indexOf(id) !== -1 || items.indexOf(id) !== -1; });
}
// The mock backup's manifest file count, in the live layout: the
// manifest itself, one file per document essential, settings as
// settings.json + voice.json + dashboard.json, chips and packs as
// their installed.json records, workflows as workflows.json. The
// live bridge counts the real files it writes; both feed the same
// githubState record the Settings page manages.
function ghMockBackupFileCount(items) {
  var count = 0; // the manifest is the index, never counted as content
  items.forEach(function (id) {
    if (id === "soul" || id === "memory") count += 1;
    else if (id === "settings") count += 3;
    else if (id === "chips" || id === "rag" || id === "workflows") count += 1;
  });
  return count;
}
var githubState = null;      // status payload (live) or simulated state (mock)
var githubFetching = false;
// The GitHub section's own tabs and live data: Overview (the
// project pulse) and Issues (the live open-issues list).
var githubTab = "overview";
var githubPulse = null;
var githubPulseFetching = false;
var githubPulseFailedAt = 0;
var githubIssuesFailedAt = 0;
var githubPulseError = "";
var githubPulseFetchedAt = null;
var githubIssues = null;
var githubIssuesFetching = false;
var githubIssuesError = "";
var githubIssuesFetchedAt = null;
// Settings' topic tabs: every setting lives under exactly one.
var settingsTab = "operator";
function settingsTabValid() { if (["operator", "persona", "appearance", "voice", "github", "system"].indexOf(settingsTab) < 0) settingsTab = "operator"; }
var githubMsg = "";          // last action's outcome line, shown on the view
var githubError = false;     // whether that line is an error
var githubDevice = null;     // {userCode, verificationUri} while a device sign-in runs
var githubPollTimer = null;
var githubPreview = null;    // last import preview {ok, changes, backedUpAt}
var githubApplied = null;    // last import apply answer
function ghSetMsg(text, isError) { githubMsg = text || ""; githubError = !!isError; }
var badgeSyncBusy = false;   // a badge-sync pull is in flight (Build 13, #136)
function mockGithubLoad() {
  var d = { repoUrl: NAVI_REPO_URL, issuesUrl: NAVI_REPO_URL + "/issues", clientId: "Ov23lijV9Q5ERi6nIAgG", clientIdConfigured: true, signedIn: false, login: "", backupRepo: "navi-backup", backupPrivate: true, backupItems: GH_ITEMS.map(function (i) { return i.id; }), intervalDays: 1, lastBackupAt: null, lastBackupFileCount: null, lastBackupResult: "", lastImportAt: null, lastImportResult: "", hasBackup: false };
  try { var raw = localStorage.getItem("navi.github"); if (raw) { var p = JSON.parse(raw); for (var k in d) if (p[k] !== undefined) d[k] = p[k]; } } catch (e) {}
  d.backupItems = ghWithCore(d.backupItems);
  return d;
}
function mockGithubSave() { try { localStorage.setItem("navi.github", JSON.stringify(githubState)); } catch (e) {} }
function fetchGithub(thenRender) {
  if (CONFIG.backend !== "live") { if (!githubState) githubState = mockGithubLoad(); if (thenRender) render(); return; }
  if (githubFetching) return;
  githubFetching = true;
  fetch("/api/github").then(function (r) { return r.json(); }).then(function (j) { githubState = j; githubFetching = false; if (thenRender) render(); }).catch(function () { githubFetching = false; if (!githubState) { githubState = mockGithubLoad(); githubState.liveError = "The bridge did not answer /api/github — a bridge from before build .35 has no GitHub endpoints. Relaunch after reinstalling."; } if (thenRender) render(); });
}
function naviToast(text) {
  // A click that cannot be carried out must say so — never a silent
  // nothing (build .49, issue #10). One small toast, bottom center,
  // spoken by aria-live, gone after a few seconds.
  var t = document.getElementById("navi-toast");
  if (!t) {
    t = document.createElement("div");
    t.id = "navi-toast";
    t.setAttribute("role", "status");
    t.setAttribute("aria-live", "polite");
    t.style.cssText = "position:fixed;left:50%;bottom:18px;transform:translateX(-50%);z-index:99;max-width:min(92vw,560px);background:#7f1d1d;color:#fecaca;border:1px solid #f87171;border-radius:6px;padding:9px 14px;font-size:13px;box-shadow:0 10px 30px rgba(0,0,0,.5)";
    document.body.appendChild(t);
  }
  t.textContent = text;
  t.style.display = "block";
  if (t._naviTimer) clearTimeout(t._naviTimer);
  t._naviTimer = setTimeout(function () { t.style.display = "none"; }, 6000);
}
function openExternal(url) {
  // Inside the standalone shell, the webview must not navigate away:
  // the shell opens the link in the real browser instead (its
  // open_external_url command takes any http(s) address) — and if
  // that invoke fails, the failure is announced, never silent.
  // Anywhere else, a new window; a refused pop-up is announced too.
  if (docsInShell()) {
    docsTauriInvoke("open_external_url", { url: url }).catch(function (err) {
      var w = null;
      try { w = window.open(url, "_blank"); } catch (e) {}
      if (!w) naviToast("That link could not be opened — the shell refused it (" + (err && err.message ? err.message : "no answer from the shell") + ") and no browser window opened. The address is: " + url + " Stand-in until this is fixed: open the Apex Web UI (http://localhost:8675) in your browser — links work there.");
    });
  } else {
    var w2 = null;
    try { w2 = window.open(url, "_blank"); } catch (e) {}
    if (!w2) naviToast("That link could not be opened — the browser blocked the new window. The address is: " + url);
  }
}
// The GitHub section's webview tabs (Roadmap, Discussions): inside
// the standalone shell the page gets its own native webview window
// (the shell's open_github_window — a top-level load, which GitHub
// serves). Outside the shell there is no window to give it and
// GitHub refuses to be framed into a page, so this is a no-op and
// the tab itself explains that and links out instead.
function openGithubPageWindow(page) {
  if (!docsInShell()) return;
  docsTauriInvoke("open_github_window", { page: page }).catch(function () {
    // Issue #10's standing rule: in the desktop app, a link
    // that fails to open ends at the one warning toast — the
    // target address plus the Web UI stand-in — and
    // openExternal is that toast. (One toast total: this
    // replaces the plainer "shell did not answer" note.)
    openExternal(page === "roadmap" ? NAVI_ROADMAP_URL : NAVI_REPO_URL + "/discussions");
  });
}
// One issue's own webview window (issue #24): clicking an issue in
// the GitHub section opens its log the same way Roadmap and
// Discussions open — a native window of the app, not the
// open_external_url route. The dashboard passes only the issue
// number; the shell validates it (digits only) and builds the
// address itself. If the shell refuses even that, fall back to
// openExternal, whose refusal toast tells the truth about it.
function openGithubIssueWindow(number, url) {
  docsTauriInvoke("open_github_window", { page: "issue:" + number }).catch(function () { openExternal(url); });
}
function ghDashboardSlice() { return { operatorName: CONFIG.operatorName || "", theme: CONFIG.theme || "", layout: CONFIG.layout || "", motion: CONFIG.motion || "", uiScale: Number(CONFIG.uiScale) || 1, operatorPicture: CONFIG.operatorPicture || "", operatorBadge: CONFIG.operatorBadge || "" }; }
function ghPost(path, body, done) {
  fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) })
    .then(function (r) { return r.json(); })
    .then(function (j) { done(j); })
    .catch(function () { ghSetMsg("The bridge did not answer " + path + " — is it running?", true); render(); });
}
function ghMapIssue(i) {
  return {
    number: i.number || 0,
    title: String(i.title || ""),
    url: String(i.html_url || i.url || ""),
    state: String(i.state || "open"),
    labels: (i.labels || []).map(function (l) { return typeof l === "string" ? l : String(l.name || ""); }).filter(Boolean),
    author: String((i.user && i.user.login) || i.author || ""),
    updatedAt: String(i.updated_at || i.updatedAt || ""),
    comments: i.comments || 0,
  };
}
function fetchGithubIssues(thenRender) {
  if (githubIssuesFetching) return;
  githubIssuesFetching = true;
  var done = function (j) {
    githubIssuesFetching = false;
    if (j && j.ok) { githubIssues = j.issues; githubIssuesError = ""; githubIssuesFetchedAt = j.fetchedAt; }
    else { githubIssuesError = (j && j.error) || "GitHub issues fetch failed."; githubIssuesFailedAt = Date.now(); }
    if (thenRender) render();
  };
  if (CONFIG.backend === "live") {
    fetch("/api/github/issues").then(function (r) { return r.json(); }).then(done).catch(function () { done({ ok: false, error: "The bridge didn't answer." }); });
  } else {
    fetch("https://api.github.com/repos/AlexKollar/Apex/issues?state=open&per_page=100&sort=updated&direction=desc")
      .then(function (r) { return r.ok ? r.json() : Promise.reject(new Error("HTTP " + r.status)); })
      .then(function (rows) { done({ ok: true, fetchedAt: new Date().toISOString(), issues: (rows || []).filter(function (i) { return !i.pull_request; }).map(ghMapIssue) }); })
      .catch(function (e) { done({ ok: false, error: "GitHub issues fetch failed: " + (e && e.message ? e.message : e) }); });
  }
}
function fetchGithubPulse(thenRender) {
  if (githubPulseFetching) return;
  githubPulseFetching = true;
  var done = function (j) {
    githubPulseFetching = false;
    if (j && j.ok && j.pulse) { githubPulse = j.pulse; githubPulseError = ""; githubPulseFetchedAt = j.fetchedAt; }
    else { githubPulseError = (j && j.error) || "GitHub pulse fetch failed."; githubPulseFailedAt = Date.now(); }
    if (thenRender) render();
  };
  if (CONFIG.backend === "live") {
    fetch("/api/github/pulse").then(function (r) { return r.json(); }).then(done).catch(function () { done({ ok: false, error: "The bridge didn't answer." }); });
  } else {
    var base = "https://api.github.com/repos/AlexKollar/Apex";
    var get = function (u) { return fetch(u).then(function (r) { return r.ok ? r.json() : Promise.reject(new Error("HTTP " + r.status)); }); };
    Promise.all([get(base), get(base + "/commits?per_page=100").catch(function () { return []; }), get(base + "/releases/latest").catch(function () { return null; }), get(base + "/issues?state=open&per_page=8").catch(function () { return []; })])
      .then(function (parts) {
        var repo = parts[0], crows = parts[1], rel = parts[2], irows = parts[3];
        done({
          ok: true,
          fetchedAt: new Date().toISOString(),
          pulse: {
            description: repo.description || "",
            stars: repo.stargazers_count || 0,
            forks: repo.forks_count || 0,
            watchers: repo.subscribers_count || repo.watchers_count || 0,
            language: repo.language || "—",
            openIssuesCount: repo.open_issues_count || 0,
            pushedAt: repo.pushed_at || "",
            defaultBranch: repo.default_branch || "main",
            commitCount: crows.length < 100 ? crows.length : null,
            latestRelease: rel && rel.tag_name ? { tag: rel.tag_name, name: rel.name || "", url: rel.html_url || "", publishedAt: rel.published_at || "" } : null,
            commits: crows.slice(0, 5).map(function (c) { return { sha: String(c.sha || "").slice(0, 7), message: String((c.commit && c.commit.message) || "").split("\n")[0], author: String((c.commit && c.commit.author && c.commit.author.name) || ""), date: String((c.commit && c.commit.author && c.commit.author.date) || ""), url: c.html_url || "" }; }),
            issues: irows.filter(function (i) { return !i.pull_request; }).slice(0, 8).map(ghMapIssue),
          },
        });
      })
      .catch(function (e) { done({ ok: false, error: "GitHub pulse fetch failed: " + (e && e.message ? e.message : e) }); });
  }
}
function ghStopPolling() { if (githubPollTimer) { clearInterval(githubPollTimer); githubPollTimer = null; } }
function ghPollOnce() {
  ghPost("/api/github/signin/poll", {}, function (j) {
    if (j && j.status === "authorized") { ghStopPolling(); githubDevice = null; ghSetMsg("Signed in as @" + (j.login || "") + " — GitHub confirmed it.", false); githubState = null; fetchGithub(true); }
    else if (j && j.status === "pending") { if (j.slowDown) { ghStopPolling(); githubPollTimer = setInterval(ghPollOnce, 10000); } }
    else { ghStopPolling(); githubDevice = null; ghSetMsg((j && j.error) || "GitHub sign-in did not complete.", true); render(); }
  });
}
function ghSignInStart() {
  ghSetMsg("", false);
  if (CONFIG.backend !== "live") {
    if (!githubState.clientId) { ghSetMsg("No GitHub OAuth client id is configured yet — sign-in can't start until one is set (see the setup steps). Viewing issues needs no sign-in at all.", true); render(); return; }
    var alpha = "ABCDEFGHJKMNPQRSTUVWXYZ23456789", raw = "";
    for (var i = 0; i < 8; i++) raw += alpha[Math.floor(Math.random() * alpha.length)];
    githubDevice = { userCode: raw.slice(0, 4) + "-" + raw.slice(4), verificationUri: "https://github.com/login/device" };
    render(); return;
  }
  ghPost("/api/github/signin/start", {}, function (j) {
    if (j && j.ok) { githubDevice = { userCode: j.userCode, verificationUri: j.verificationUri }; ghStopPolling(); githubPollTimer = setInterval(ghPollOnce, Math.max(5, j.interval || 5) * 1000); }
    else ghSetMsg((j && j.error) || "GitHub did not start a sign-in.", true);
    render();
  });
}
function ghSignInCompleteMock() {
  githubState.signedIn = true; githubState.login = "operator-sim"; mockGithubSave();
  githubDevice = null; ghSetMsg("Signed in as @operator-sim (simulated — mock mode has no real GitHub behind it).", false); render();
}
function ghSignOut() {
  ghStopPolling(); githubDevice = null; githubPreview = null; githubApplied = null;
  if (CONFIG.backend !== "live") { githubState.signedIn = false; githubState.login = ""; mockGithubSave(); ghSetMsg("Signed out. Navi herself never needed the sign-in.", false); render(); return; }
  ghPost("/api/github/signout", {}, function () { githubState = null; ghSetMsg("Signed out — the token file is deleted. Navi herself never needed the sign-in.", false); fetchGithub(true); });
}
function ghBucketLine(label, b) {
  if (!b) return "<p class='mono' style='color:var(--dim);font-size:11px'>" + label + " — no GitHub response has reported limits yet</p>";
  var rs = "";
  if (b.reset) { var d = new Date(b.reset * 1000); rs = " · resets " + ("0" + d.getHours()).slice(-2) + ":" + ("0" + d.getMinutes()).slice(-2); }
  return "<p class='mono' style='color:var(--dim);font-size:11px'>" + label + " — " + (b.remaining === null || b.remaining === undefined ? "?" : b.remaining) + " of " + b.limit + " left" + rs + "</p>";
}
function ghSaveBackupSettings() {
  var repoEl = document.getElementById("gh-repo");
  var visEl = document.getElementById("gh-vis");
  var intEl = document.getElementById("gh-interval");
  var items = [];
  GH_ITEMS.forEach(function (it) { var c = document.getElementById("gh-item-" + it.id); if (c && c.checked) items.push(it.id); });
  items = ghWithCore(items);
  var patch = { backup_repo: repoEl ? repoEl.value.trim() : "navi-backup", backup_private: visEl ? visEl.value === "private" : true, backup_items: items, backup_interval_days: intEl ? parseInt(intEl.value, 10) : 1 };
  if (CONFIG.backend !== "live") { githubState.backupRepo = patch.backup_repo || "navi-backup"; githubState.backupPrivate = patch.backup_private; githubState.backupItems = items; githubState.intervalDays = patch.backup_interval_days; mockGithubSave(); ghSetMsg("Backup settings saved (simulated state — mock mode).", false); render(); return; }
  ghPost("/api/github/settings", patch, function (j) { if (j && j.ok !== false && j.backupRepo !== undefined) { githubState = j; ghSetMsg("Backup settings saved.", false); } else ghSetMsg((j && j.error) || "The settings were refused.", true); render(); });
}
function ghBackupNow() {
  ghSetMsg("Backing up…", false); githubApplied = null;
  if (CONFIG.backend !== "live") {
    if (!githubState.signedIn) { ghSetMsg("Backups need a GitHub sign-in first — Navi herself doesn't, but writing to GitHub does.", true); render(); return; }
    if (!githubState.backupItems.length) { ghSetMsg("Nothing is ticked — a backup with no essentials in it would restore an empty Navi.", true); render(); return; }
    var snap = { takenAt: new Date().toISOString(), items: githubState.backupItems.slice(), soul: CONFIG.soul || "", memory: CONFIG.memory || "", operatorName: CONFIG.operatorName || "", theme: CONFIG.theme || "", layout: CONFIG.layout || "", motion: CONFIG.motion || "", uiScale: Number(CONFIG.uiScale) || 1 };
    try { localStorage.setItem("navi.github.snapshot", JSON.stringify(snap)); } catch (e) {}
    githubState.hasBackup = true; githubState.lastBackupAt = snap.takenAt;
    githubState.lastBackupFileCount = ghMockBackupFileCount(snap.items);
    githubState.lastBackupResult = snap.items.length + " essentials written to " + githubState.login + "/" + githubState.backupRepo + (githubState.backupPrivate ? " (private)" : " (PUBLIC)") + " — simulated in mock mode";
    mockGithubSave(); ghSetMsg(githubState.lastBackupResult + ".", false); render(); return;
  }
  render();
  ghPost("/api/github/backup", { dashboard: ghDashboardSlice() }, function (j) {
    if (j && j.ok) ghSetMsg("Backup done — " + j.result + ".", false); else ghSetMsg((j && j.error) || "The backup failed.", true);
    githubState = null; fetchGithub(true);
  });
}
function ghImportPreview() {
  ghSetMsg("Reading the backup…", false); githubApplied = null;
  if (CONFIG.backend !== "live") {
    var snap = null;
    try { snap = JSON.parse(localStorage.getItem("navi.github.snapshot") || "null"); } catch (e) {}
    if (!snap) { githubPreview = { ok: false, error: "No backup is on record yet — run a backup first, then import has something to restore.", changes: [] }; ghSetMsg("", false); render(); return; }
    var changes = [];
    var labelOf = {}; GH_ITEMS.forEach(function (i) { labelOf[i.id] = i.label; });
    snap.items.forEach(function (id) {
      if (id === "soul") changes.push({ item: id, label: labelOf[id], detail: (CONFIG.soul || "") === snap.soul ? "Already matches the backup — nothing to change." : "Would replace the current Soul (" + (CONFIG.soul || "").length + " characters) with the backup's (" + snap.soul.length + " characters).", willChange: (CONFIG.soul || "") !== snap.soul });
      else if (id === "memory") changes.push({ item: id, label: labelOf[id], detail: (CONFIG.memory || "") === snap.memory ? "Already matches the backup — nothing to change." : "Would replace the current Memory (" + (CONFIG.memory || "").length + " characters) with the backup's (" + snap.memory.length + " characters).", willChange: (CONFIG.memory || "") !== snap.memory });
      else if (id === "settings") { var d = []; if ((CONFIG.operatorName || "") !== snap.operatorName) d.push("operator name → " + (snap.operatorName || "Operator")); if ((CONFIG.theme || "") !== snap.theme) d.push("theme"); if ((CONFIG.layout || "") !== (snap.layout || "")) d.push("layout"); if ((CONFIG.motion || "") !== (snap.motion || "")) d.push("motion"); if ((Number(CONFIG.uiScale) || 1) !== (Number(snap.uiScale) || 1)) d.push("UI scale"); changes.push({ item: id, label: labelOf[id], detail: d.length ? "Would restore: " + d.join(" · ") + "." : "Already matches the backup — nothing to change.", willChange: !!d.length }); }
      else changes.push({ item: id, label: labelOf[id], detail: "Simulated in mock mode — the live bridge and the bench in the Apex space restore this essential in full.", willChange: false });
    });
    githubPreview = { ok: true, changes: changes, backedUpAt: snap.takenAt }; ghSetMsg("", false); render(); return;
  }
  render();
  ghPost("/api/github/import/preview", { dashboard: ghDashboardSlice() }, function (j) { githubPreview = j; ghSetMsg("", false); render(); });
}
function ghImportApply() {
  ghSetMsg("Importing…", false);
  if (CONFIG.backend !== "live") {
    var snap = null;
    try { snap = JSON.parse(localStorage.getItem("navi.github.snapshot") || "null"); } catch (e) {}
    if (!snap) { ghSetMsg("No backup is on record yet.", true); render(); return; }
    if (snap.items.indexOf("soul") !== -1) CONFIG.soul = snap.soul;
    if (snap.items.indexOf("memory") !== -1) CONFIG.memory = snap.memory;
    if (snap.items.indexOf("settings") !== -1) { CONFIG.operatorName = snap.operatorName; if (["dark", "light", "contrast", "nord", "terminal", "glass", "nocturne", "abyss", "casefile", "bloodmoon", "phosphor", "lcars"].indexOf(snap.theme) !== -1) CONFIG.theme = snap.theme; if (snap.layout === "lcars" || snap.layout === "standard") CONFIG.layout = snap.layout; if (snap.motion === "potato" || snap.motion === "full") CONFIG.motion = snap.motion; if ([1, 1.1, 1.25, 1.5].indexOf(Number(snap.uiScale)) !== -1) CONFIG.uiScale = Number(snap.uiScale); }
    saveCfg(); applyTheme(); applyMotion(); applyLayout(); applyUiScale(); refreshSeedGreeting();
    githubState.lastImportAt = new Date().toISOString();
    githubState.lastImportResult = "Imported the (simulated) backup from " + String(snap.takenAt).slice(0, 10) + " — documents and dashboard settings restored.";
    mockGithubSave();
    githubApplied = { ok: true, result: githubState.lastImportResult };
    ghSetMsg(githubState.lastImportResult, false); render(); return;
  }
  render();
  ghPost("/api/github/import/apply", {}, function (j) {
    if (j && j.ok) {
      githubApplied = j;
      if (j.dashboard) {
        if (typeof j.dashboard.operatorName === "string") CONFIG.operatorName = j.dashboard.operatorName;
        if (["dark", "light", "contrast", "nord", "terminal", "glass", "nocturne", "abyss", "casefile", "bloodmoon", "phosphor", "lcars"].indexOf(j.dashboard.theme) !== -1) CONFIG.theme = j.dashboard.theme;
        if (j.dashboard.layout === "lcars" || j.dashboard.layout === "standard") CONFIG.layout = j.dashboard.layout;
        if (j.dashboard.motion === "potato" || j.dashboard.motion === "full") CONFIG.motion = j.dashboard.motion;
        if ([1, 1.1, 1.25, 1.5].indexOf(Number(j.dashboard.uiScale)) !== -1) CONFIG.uiScale = Number(j.dashboard.uiScale);
        if (typeof j.dashboard.operatorPicture === "string" && /^data:image\//.test(j.dashboard.operatorPicture) && j.dashboard.operatorPicture.length <= 420000) CONFIG.operatorPicture = j.dashboard.operatorPicture;
        if (typeof j.dashboard.operatorBadge === "string" && /^data:image\//.test(j.dashboard.operatorBadge) && j.dashboard.operatorBadge.length <= 420000) CONFIG.operatorBadge = j.dashboard.operatorBadge;
        saveCfg(); applyTheme(); applyMotion(); applyLayout(); applyUiScale(); refreshSeedGreeting();
      }
      // Her persona may have been restored under us — re-read it.
      fetch("/api/persona").then(function (r) { return r.json(); }).then(function (p) { if (p.soul) CONFIG.soul = p.soul; if (p.memory) CONFIG.memory = p.memory; }).catch(function () {});
      ghSetMsg(j.result, false);
    } else ghSetMsg((j && j.error) || "The import failed.", true);
    githubState = null; fetchGithub(true);
  });
}
var naviThinking = false;    // typing indicator while a reply is in flight
var chatFocusWanted = false; // a chat send asked for the composer to take focus back
var studioDraft = null;      // {file|null, name, description, code, output, files, active, folds, snapshot} in the editor
var docsPage = "getting-started";  // the Documentation wiki's open page
// Docs pop-out: the wiki's home is the Docs view — the main
// documentation window; popping out opens the documentation in
// another window entirely — a separate window holding this same
// dashboard flagged #popdocs, which boots straight into a
// docs-only window. Inside the standalone shell the webview
// refuses window.open, so the dashboard asks the shell itself for
// its native second window (the open_docs_window command in
// shell/src/main.rs); in a plain browser, window.open is that
// second window. There is no in-page panel: if a browser refuses
// the new window, the Docs view says so and offers a direct
// new-window link instead of pretending a panel was a pop-out.
// The two windows share the open page through localStorage
// ("navi.docs.page") and the popped window asks to pop back with
// "navi.docs.dock".
var DOCS_POP = false;
try { DOCS_POP = location.hash.indexOf("popdocs") !== -1 || new URLSearchParams(location.search).has("popdocs"); } catch (e) {}
if (DOCS_POP) { try { var savedDocsPage = localStorage.getItem("navi.docs.page"); if (savedDocsPage) docsPage = savedDocsPage; } catch (e) {} }
var docsPopWin = null;       // the browser pop-out window, while it lives
var docsShellOut = false;    // the standalone shell's native docs window is open
var docsPopBlocked = false;  // a pop-out was refused — no window opened at all
var docsDockSeen = "";       // last "navi.docs.dock" value the main window acted on
function docsInShell() { try { return !!(window.__TAURI__ && window.__TAURI__.core && typeof window.__TAURI__.core.invoke === "function"); } catch (e) { return false; } }
function docsTauriInvoke(cmd, args) { try { return window.__TAURI__.core.invoke(cmd, args); } catch (e) { return Promise.reject(e); } }
function docsPopUrl() { return location.href.split("#")[0].split("?")[0] + "#popdocs"; }
function docsPageTitle(id) { var hit = DOC_PAGES.filter(function (x) { return x.id === id; })[0]; return hit ? hit.title : id; }
// The wiki's pages, grouped by topic: the page list renders these
// topic headings with their pages nested under them, in this
// order — start first, everyday talking next, making things after
// that, then settings and help. Every page appears in exactly one
// topic. Each topic is a dropdown: its heading expands and
// collapses its pages with a smooth animation. Topics start open;
// navigating to a page (a cross-link, the other window) opens its
// topic, so the list never greets the operator fully shut.
var DOC_GROUPS = [
  { id: "topic-start", title: "Getting started", pages: ["getting-started"] },
  { id: "topic-talking", title: "Talking to Navi", pages: ["chat-cli", "working-day", "voice"] },
  { id: "topic-chips", title: "Chips", pages: ["chips-guide", "navi-functions", "studio-editor", "chips-publishing", "chipdeck"] },
  { id: "topic-knowledge", title: "Knowledge & RAG", pages: ["rag", "vault", "case-files", "threat-intelligence"] },
  { id: "topic-settings", title: "Settings & updates", pages: ["settings", "dhd", "mesh", "federation", "fleet-answers", "fedichat", "guardian", "github", "installing", "server-mode", "headless-autostart"] },
  { id: "topic-help", title: "Help", pages: ["troubleshooting", "glossary"] }
];
var docsTopicClosed = {};    // topic id -> true while the operator has it collapsed
var docsPageTopicSeen = "";  // last open page, for topic auto-open on navigation
function docsEnsureTopicOpen() {
  if (docsPageTopicSeen === docsPage) return;
  docsPageTopicSeen = docsPage;
  DOC_GROUPS.forEach(function (g) { if (g.pages.indexOf(docsPage) !== -1) delete docsTopicClosed[g.id]; });
}
// Topic dropdowns: flip the state and the DOM in place — no
// render, so the grid-rows glide actually animates instead of
// the rebuilt list snapping to its final shape. Shared by the
// Docs view's wiring and the popped window's own wiring.
function wireDocTopics(rootEl) {
  rootEl.querySelectorAll("[data-doctopic]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-doctopic");
    var open = !!docsTopicClosed[id];
    if (open) delete docsTopicClosed[id]; else docsTopicClosed[id] = true;
    var box = rootEl.querySelector("[data-doctopic-pages='" + id + "']");
    if (box) box.classList.toggle("closed", !open);
    b.setAttribute("aria-expanded", open ? "true" : "false");
    var title = b.querySelector("span") ? b.querySelector("span").textContent : "topic";
    b.setAttribute("aria-label", (open ? "Collapse" : "Expand") + " the " + title + " topic");
  }; });
}
// The menu's scroll position survives re-renders (issue #119):
// captured from the live menu before any rebuild of the Docs
// view or the popped window, restored after — and the active
// page is scrolled into view when a navigation leaves it
// outside the menu's visible stretch (instantly under reduced
// motion or Potato, a gentle glide otherwise).
var docsNavScrollTop = 0;
function docsCaptureNav() {
  var n = mainEl.querySelector(".docs-wiki-nav");
  if (n) docsNavScrollTop = n.scrollTop;
}
function docsRestoreNav() {
  var n = mainEl.querySelector(".docs-wiki-nav");
  if (!n) return;
  n.scrollTop = docsNavScrollTop;
  var act = n.querySelector("button.act[data-docpage]");
  if (!act) return;
  var nr = n.getBoundingClientRect(), ar = act.getBoundingClientRect();
  if (ar.top < nr.top || ar.bottom > nr.bottom) {
    var calm = CONFIG.motion === "potato" || (window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches);
    act.scrollIntoView({ block: "nearest", behavior: calm ? "auto" : "smooth" });
  }
}
function docsWikiHtml() {
  docsEnsureTopicOpen();
  var dp = DOC_PAGES.filter(function (x) { return x.id === docsPage; })[0] || DOC_PAGES[0];
  var nav = DOC_GROUPS.map(function (g, gi) {
    var open = !docsTopicClosed[g.id];
    var items = g.pages.map(function (pid) {
      var x = DOC_PAGES.filter(function (y) { return y.id === pid; })[0];
      if (!x) return "";
      return "<li><button class='" + (x.id === dp.id ? "act" : "ghost") + "' style='width:100%;text-align:left' data-docpage='" + x.id + "' aria-label='Open the " + esc(x.title) + " documentation page'" + (x.id === dp.id ? " aria-current='page'" : "") + ">" + esc(x.title) + "</button></li>";
    }).join("");
    return "<div style='" + (gi === 0 ? "" : "margin-top:12px") + "'><button type='button' class='doc-topic-head' data-doctopic='" + g.id + "' aria-expanded='" + (open ? "true" : "false") + "' aria-label='" + (open ? "Collapse" : "Expand") + " the " + esc(g.title) + " topic'><span>" + esc(g.title) + "</span><span class='doc-chev' aria-hidden='true'>▾</span></button>"
      + "<div class='doc-topic-pages" + (open ? "" : " closed") + "' data-doctopic-pages='" + g.id + "'><div class='doc-topic-inner'><ul style='list-style:none;margin:4px 0 0;padding:0;display:flex;flex-direction:column;gap:4px'>" + items + "</ul></div></div></div>";
  }).join("");
  // The phone page picker (build .66): one grouped select naming
  // every page under its topic — rendered always, shown only at
  // phone widths, where the topic list beside it hides (the CSS
  // owns both halves). Its value is the open page; choosing one
  // navigates exactly like the list's buttons do.
  var picker = "<select id='docs-picker' class='docs-picker' aria-label='Choose a documentation page'>" + DOC_GROUPS.map(function (g) {
    return "<optgroup label='" + esc(g.title) + "'>" + g.pages.map(function (pid) {
      var x = DOC_PAGES.filter(function (y) { return y.id === pid; })[0];
      return x ? "<option value='" + x.id + "'" + (x.id === dp.id ? " selected" : "") + ">" + esc(x.title) + "</option>" : "";
    }).join("") + "</optgroup>";
  }).join("") + "</select>";
  return picker + "<div class='docs-panes'><nav class='docs-wiki-nav' aria-label='Documentation pages'>" + nav + "</nav>"
    + "<div class='docs-wiki-body'>" + dp.html
    + "<p style='color:var(--dim)'>See also: " + dp.related.map(function (r) { return "<button class='ghost' data-docpage='" + r + "' aria-label='Open the " + esc(docsPageTitle(r)) + " documentation page'>" + esc(docsPageTitle(r)) + "</button>"; }).join(" ") + "</p></div></div>";
}
function docsPoppedOut() { return docsShellOut || !!(docsPopWin && !docsPopWin.closed); }
function popOutDocs() {
  try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {}
  docsPopBlocked = false;
  if (docsInShell()) {
    // The shell's webview will not honor window.open — ask the
    // shell for its own second window instead. That window is a
    // real window: separate, movable, closable on its own.
    if (docsShellOut) { docsTauriInvoke("open_docs_window").catch(function () {}); return; }
    docsShellOut = true;
    docsTauriInvoke("open_docs_window").then(function () { docsShellOut = true; }).catch(function () { docsShellOut = false; docsPopBlocked = true; render(); });
    return;
  }
  if (docsPopWin && !docsPopWin.closed) { docsPopWin.focus(); return; }
  docsPopWin = window.open(docsPopUrl(), "navi-docs", "width=900,height=1020,popup=1");
  if (!docsPopWin) { docsPopBlocked = true; return; }
  docsPopWin.focus();
}
// Pop back in (main window side): close the popped window — the
// shell's own window through the shell, a browser window through
// its handle — take back the page it was left on, and let the
// Docs view show the wiki again.
function dockDocs() {
  if (docsShellOut && docsInShell()) docsTauriInvoke("close_docs_window").catch(function () {});
  docsShellOut = false;
  if (docsPopWin && !docsPopWin.closed) docsPopWin.close();
  docsPopWin = null;
  docsPopBlocked = false;
  try { var p = localStorage.getItem("navi.docs.page"); if (p) docsPage = p; } catch (e) {}
  render();
}
// The Docs view's wiki block: its home, with its way out — and,
// while the docs are out, its placeholder and way back. When a
// pop-out was refused outright, a plain note under the wiki says
// so and carries a direct link that opens the documentation in a
// new window on the operator's own click.
function docsHomeHtml() {
  var h = "<div class='docs-frame'><div style='display:flex;align-items:flex-start;justify-content:space-between;gap:8px;flex-wrap:wrap'><p style='color:var(--dim);max-width:640px;margin:0'>One page per topic, written plainly — short steps, in order. This view is the wiki's home: pop it out of here when you want it beside your work, and pop it back in when you're done.</p>";
  if (docsPoppedOut()) {
    h += "</div><div class='card' style='margin-top:10px'><span style='color:var(--dim)'>The documentation is popped out in its own window, open at <b>" + esc(docsPageTitle(docsPage)) + "</b> — read it side-by-side with the chat or the editor.</span><br><button class='ghost' id='docs-pop-in' style='margin-top:6px' aria-label='Pop the documentation back into the Docs view'>⇙ Pop back in</button></div>";
  } else {
    h += "<button class='ghost' id='docs-pop-out' aria-label='Pop the documentation out of the Docs view into its own window'>⧉ Pop out</button></div>"
      + docsWikiHtml();
    if (docsPopBlocked) {
      h += "<div class='card docs-blocked' style='margin-top:10px;border-color:#f5b544;color:#f5b544'>The pop-out was blocked — the documentation did not open anywhere. <a class='ghost' style='border-color:#f5b544;color:#f5b544' href='" + esc(docsPopUrl()) + "' target='_blank' rel='noopener' aria-label='Open the documentation in a new window'>Open the documentation in a new window</a> — the same wiki, in a window of its own. Stand-in until this is fixed: open the Apex Web UI (http://localhost:8675) in your browser — links work there.</div>";
    }
  }
  return h + "</div>";
}
// The popped window itself: no nav, no status column — the wiki,
// a header, and the way back. Pop back in leaves the page in
// localStorage for the main window and asks it to dock, then
// closes; if the host refuses the close, the note says the rest.
function renderDocsPop() {
  docsCaptureNav();
  var pbar = document.getElementById("presence-bar");
  if (pbar) pbar.style.display = "none";
  if (navEl) navEl.style.display = "none";
  if (statusEl) statusEl.style.display = "none";
  if (toggleBtn) toggleBtn.style.display = "none";
  if (appEl) appEl.style.display = "block";
  document.title = "Apex — Documentation";
  mainEl.classList.remove("chat-view");
  mainEl.classList.add("docs-pop");
  mainEl.style.height = "";
  mainEl.innerHTML = "<div style='display:flex;align-items:center;justify-content:space-between;gap:8px;flex-wrap:wrap'><h3 style='margin:0'>Documentation</h3><button class='ghost' id='docs-pop-back' aria-label='Pop the documentation back into the Docs view'>⇙ Pop back in</button></div>"
    + "<p id='docs-pop-note' style='color:var(--dim);font-size:12px'>Popped out of the Docs view — the Docs view in the main window is holding your place.</p>"
    + "<div class='docs-frame'>" + docsWikiHtml() + "</div>";
  docsRestoreNav();
  mainEl.querySelectorAll("[data-docpage]").forEach(function (b) { b.onclick = function () { docsPage = b.getAttribute("data-docpage") || "getting-started"; try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {} renderDocsPop(); }; });
  wireDocTopics(mainEl);
  var popPicker = document.getElementById("docs-picker");
  if (popPicker) popPicker.onchange = function () { docsPage = popPicker.value || "getting-started"; try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {} renderDocsPop(); };
  var back = document.getElementById("docs-pop-back");
  if (back) back.onclick = function () {
    try { localStorage.setItem("navi.docs.page", docsPage); localStorage.setItem("navi.docs.dock", String(Date.now())); } catch (e) {}
    // In the shell, this window is the shell's own — ask it to
    // close this window; window.close() alone is refused there.
    if (docsInShell()) docsTauriInvoke("close_docs_window").catch(function () {});
    window.close();
    setTimeout(function () { var n = document.getElementById("docs-pop-note"); if (n) n.textContent = "Back in the Docs view in the main window — you can close this window now."; }, 400);
  };
}
// Cross-window manners: follow the page the other window is on,
// and dock when the popped window asks.
window.addEventListener("storage", function (e) {
  if (DOCS_POP) return;
  if (e.key === "navi.docs.page" && e.newValue) {
    docsPage = e.newValue;
    if (view === "Docs") render();
  } else if (e.key === "navi.docs.dock") {
    docsDockSeen = e.newValue || "";
    dockDocs();
  }
});
// Storage events do not reliably cross between the shell's two
// webview windows, so the main window also polls: a dock request
// left in localStorage is honored within the second, and a shell
// docs window the operator closed on its own docks the wiki here.
setInterval(function () {
  if (DOCS_POP) return;
  try {
    var dockReq = localStorage.getItem("navi.docs.dock") || "";
    if (dockReq && dockReq !== docsDockSeen) {
      docsDockSeen = dockReq;
      if (docsPoppedOut()) { dockDocs(); return; }
    }
  } catch (e) {}
  if (docsShellOut && docsInShell()) {
    docsTauriInvoke("docs_window_open").then(function (open) {
      if (!open && docsShellOut) {
        docsShellOut = false;
        try { var p = localStorage.getItem("navi.docs.page"); if (p) docsPage = p; } catch (e) {}
        if (view === "Docs") render();
      }
    }).catch(function () {});
  }
}, 800);
var studioFocus = false;     // studio focus mode (Build 13, #146 — the Vault's #135 idiom; replaces the retired editor fullscreen)
var studioPane = "code";     // phone pane state (Build 13, #146): code | test — Face is the collapsible strip, not a pane
var studioFaceOpen = false;  // phone face-strip expanded state (Build 13, #146)
var studioRunOut = "";       // last phone Run result, shown in the Test pane (Build 13, #146)
var studioWrap = false;      // studio editor word wrap
var studioZoom = (window.matchMedia && window.matchMedia("(pointer: coarse)").matches) ? 16 : 12.5; // studio editor font size (px); touch starts at 16 so focusing the editor never triggers the browser's focus zoom
var studioExtras = [];       // studio editor extra caret offsets (multi-cursor)
var studioPal = false;       // studio command palette open
var studioPalQuery = "";
var studioFindAll = "";      // find-across-files query
var studioFindAllResults = null;
var chipExp = {};            // folder chip file -> export result {file, files} | {error}
var studioSaveTimer = null;  // autosave debounce
var studioRename = null;     // file path being renamed in the studio file list
var studioRenameText = "";
var STUDIO_COMMANDS = [
  ["save", "Save / forge the chip"], ["foldall", "Fold all blocks"], ["unfoldall", "Unfold all blocks"],
  ["duplicate", "Duplicate line"], ["moveup", "Move line up"], ["movedown", "Move line down"],
  ["comment", "Toggle comment"], ["goto", "Go to line…"], ["find", "Find in this file"],
  ["findall", "Find in all files"], ["wrap", "Toggle word wrap"], ["zoomin", "Zoom in"],
  ["zoomout", "Zoom out"], ["focus", "Toggle focus mode"], ["clearcursors", "Clear extra cursors"]
];
var STUDIO_SNIPPETS = {
  run: "def run(arguments=None):\n  navi_instance = navi_internal.navi_instance\n  \n",
  import: "import navi_internal\n",
  header: "command: str = \"\"\nuse: str = \"\"\n",
  print: "navi_instance.print_message(\"\")",
  llm: "navi_instance.llm_chat(\"\")",
  user: "navi_instance.get_user()"
};
var studioMsg = "";
var studioFind = "";         // find-bar query in the studio editor
var wfData = null;           // live: last /api/workflows answer {workflows, chips}
var wfRuns = {};             // live: workflow id -> last run result
var wfDraftSteps = [];       // live: builder chain (chip files)
var mockWorkflows = [];      // mock: workflows in localStorage
var mockWfRuns = {};         // mock: workflow id -> last run result
var mockWfSteps = [];        // mock: builder chain (CHIP_DEFS ids)
try { mockWorkflows = JSON.parse(localStorage.getItem("navi.workflows") || "[]"); } catch (e) {}
function persistWorkflows() { try { localStorage.setItem("navi.workflows", JSON.stringify(mockWorkflows)); } catch (e) {} }
// The seeded greeting's fixed body — everything after its
// Hello_<name>() opening. The opening carries the name the operator
// selected in Settings (opName(), defaulting to Operator): the name
// on the UI is the operator's own choice, never the retired
// Hello_Name() placeholder.
var GREETING_TAIL = "() — hey, I'm Navi. I live on this machine and I work through it for real: ask me to look at a directory and I'll run the command instead of guessing. The tabs on the left are everything I can do — chips, memory, your documents, the shell. What are we getting into?";
function seedGreetingText() { return "Hello_" + opName() + GREETING_TAIL; }
// Re-renders the main session's seeded greeting from the current
// selection. A stored opening message counts as the untouched seed
// when it is the template rendered under any name — the operator
// cannot edit messages, so that form is the seed and only the seed;
// anything else is history and stays exactly as it was.
function refreshSeedGreeting() {
  var c = chats && chats[0];
  var first = c && c.messages && c.messages[0];
  if (!first || first.who !== "NAVI" || typeof first.text !== "string") return;
  if (first.text.indexOf("Hello_") !== 0) return;
  if (first.text.slice(-GREETING_TAIL.length) !== GREETING_TAIL) return;
  var want = seedGreetingText();
  if (first.text !== want) { first.text = want; persist(); }
}
function seedChats() {
  chats = [{ id: 1, title: "Main session", messages: [{ who: "NAVI", text: seedGreetingText(), exec: null }] }];
  activeChatId = 1;
  messages = chats[0].messages;
}
try {
  var savedChats = JSON.parse(localStorage.getItem("navi.chats") || "null");
  if (savedChats && savedChats.length) { chats = savedChats; activeChatId = Number(localStorage.getItem("navi.activeChat") || savedChats[0].id); }
  memory = JSON.parse(localStorage.getItem("navi.memory") || "[]");
  docs = JSON.parse(localStorage.getItem("navi.docs") || "null") || [];
} catch (e) {}
// Memory entries (build .81, issue #72): an entry is an object
// now — {t: the fact, o: how it got there: manual / trigger /
// autonomous / correction} — so the two Settings files (Operator
// for manual / trigger, Persona for the rest) can say where each
// entry came from and a chat receipt's Undo can find its entry.
// Entries saved by earlier builds are plain strings; they
// normalize to "manual" on load, which is exactly what they were.
memory = (memory || []).map(function (m) {
  if (typeof m === "string") return { t: m, o: "manual" };
  return { t: String((m && m.t) || ""), o: (m && m.o) || "manual" };
}).filter(function (m) { return !!m.t; });
// Ask-first memory proposals wait here (and mirror onto their
// chat receipts) until the operator approves or dismisses them.
var memoryProposals = [];
try { memoryProposals = JSON.parse(localStorage.getItem("navi.memoryProposals") || "[]"); } catch (e) { memoryProposals = []; }
function persistMemoryProposals() { try { localStorage.setItem("navi.memoryProposals", JSON.stringify(memoryProposals)); } catch (e) {} }
if (!chats) seedChats();
(function () {
  var act = chats.filter(function (c) { return c.id === activeChatId; })[0] || chats[0];
  activeChatId = act.id; messages = act.messages;
})();
/* ------------- Server-chats state (declared early, #182) ------------- */
// v0.4.6e (#182): these three lived BELOW the startup block
// that follows — in the "Server-side chats" section, some 40
// lines after the refreshSeedGreeting() call below. var hoists
// the NAME, not the assignment: on any load where the stored
// greeting re-rendered (or the retired seed docs were stripped),
// refreshSeedGreeting → persist → stampChatActivity reached
// chatStampSeen while it was still undefined and the whole
// script died on the TypeError — the dashboard blank on every
// reload after the first visit, exactly as field-reported.
// The state now stands above every call that can reach it.
var chatsServerReady = false;  // the store has answered once
var chatsSyncTimer = null;
var chatStampSeen = {};        // chat id -> last seen message count

// A greeting persisted by an earlier build — under the retired
// Hello_Name() placeholder or a previously selected name — is still
// the untouched seed, so it re-renders under the current selection
// on load; the Settings save does the same the moment it lands.
refreshSeedGreeting();
// The two sample documents earlier builds seeded into the RAG
// index are retired: the operator asked for them out, and the
// index now holds only knowledge packs he installs or forges (the
// RAG tab). Strip exact copies of the old seeds — title AND body,
// so a document he added or edited himself never matches — from
// installs that already persisted them, and save the cleaned list
// so they stay gone.
(function () {
  var SEEDS = [
    { title: "incident-playbook.txt", body: "Incident playbook: isolate the host, snapshot memory, pull authentication logs for the last 24 hours, check scheduled tasks and persistence mechanisms." },
    { title: "threat-summary.txt", body: "Threat summary: phishing is the top initial-access vector. Credential reuse is second. Patch latency on edge devices is third." }
  ];
  var kept = docs.filter(function (d) { return !SEEDS.some(function (s) { return d.title === s.title && d.body === s.body; }); });
  if (kept.length !== docs.length) { docs = kept; persist(); }
})();
function activeChat() { return chats.filter(function (c) { return String(c.id) === String(activeChatId); })[0] || chats[0]; }
function persist() {
  stampChatActivity();
  try {
    localStorage.setItem("navi.memory", JSON.stringify(memory));
    localStorage.setItem("navi.docs", JSON.stringify(docs));
    localStorage.setItem("navi.chats", JSON.stringify(chats));
    localStorage.setItem("navi.activeChat", String(activeChatId));
  } catch (e) {}
  // Build 12 (#139): the localStorage copy is a cache now — the
  // store of record is the bridge's, and every persist writes
  // through to it (debounced; a no-op until the store answers).
  chatsWriteThrough();
  // v0.5.1e (#256): the seat writes through the same way — her
  // memory entries, the operator's name, picture and badge are
  // the bridge's record, and a change (or a forget) rides up on
  // the persist that made it, not on the next session's note.
  seatWriteThrough();
}

/* ---------------- Server-side chats (Build 12, #139) ---------------- */
// The store of record for chats now lives on the bridge, in plain
// files under config/chats/ — the window hydrates from it on load,
// writes through on every persist, and keeps localStorage as a
// cache and an offline fallback only. Any door into the same
// install (this window, a DHD session, another browser) therefore
// opens the same threads. Uploads follow chats: their index rides
// in the same sync payload.
function stampChatActivity() {
  // #182: belt and braces — the declarations stand above the
  // startup block now, but this function is on persist()'s
  // path and persist() is called from all over the file; an
  // uninitialized map must never be able to kill the page
  // again, whatever order future edits leave behind.
  if (typeof chatStampSeen === "undefined" || !chatStampSeen) chatStampSeen = {};
  chats.forEach(function (c) {
    var n = (c.messages || []).length;
    if (chatStampSeen[c.id] !== undefined && chatStampSeen[c.id] !== n) c.updatedAt = Date.now();
    chatStampSeen[c.id] = n;
    if (!c.createdAt) c.createdAt = c.updatedAt || Date.now();
    if (!c.updatedAt) c.updatedAt = c.createdAt;
  });
}
function chatsSyncPayload() {
  return { chats: chats, uploads: uploads };
}
function chatsWriteThroughNow() {
  if (CONFIG.backend !== "live" || !chatsServerReady) return;
  fetch("/api/chats/sync", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(chatsSyncPayload()) }).catch(function () {});
}
function chatsWriteThrough() {
  if (CONFIG.backend !== "live" || !chatsServerReady) return;
  if (chatsSyncTimer) clearTimeout(chatsSyncTimer);
  chatsSyncTimer = setTimeout(function () { chatsSyncTimer = null; chatsWriteThroughNow(); }, 350);
}
function hydrateChatsFromServer() {
  if (CONFIG.backend !== "live") return;
  fetch("/api/chats").then(function (r) { return r.json(); }).then(function (j) {
    if (!j || !Array.isArray(j.chats)) return;
    chatsServerReady = true;
    if (j.chats.length) {
      chats = j.chats.map(function (c) {
        var n = Number(c.id);
        if (!isNaN(n) && String(n) === String(c.id)) c.id = n;
        return c;
      });
      if (Array.isArray(j.uploads)) uploads = j.uploads;
      var act = chats.filter(function (c) { return c.id === activeChatId; })[0] || chats[0];
      activeChatId = act.id; messages = act.messages;
      stampChatActivity();
      persist();
      if (view === "Chat") render();
    } else {
      // An empty store: whatever this browser already holds (the
      // one-time migration from the localStorage years) becomes
      // the store's seed — imported, never discarded.
      chatsWriteThroughNow();
    }
  }).catch(function () {});
}

/* ------------- Account sync via the Operator Badge (#136) ------------- */
// The Badge is a pointer and a trigger, never an authenticator:
// slotting it asks the bridge to pull the latest GitHub backup and
// lay the per-item preview on a live operation card in Chat. The
// GitHub sign-in is the only key; signed out, the flow stops at
// its first step and fetches nothing. Nothing is applied until
// the operator confirms on the card.
function adoptDashboardSlice(d) {
  if (!d) return;
  if (typeof d.operatorName === "string") CONFIG.operatorName = d.operatorName;
  if (["dark", "light", "contrast", "nord", "terminal", "glass", "nocturne", "abyss", "casefile", "bloodmoon", "phosphor", "lcars"].indexOf(d.theme) !== -1) CONFIG.theme = d.theme;
  if (d.layout === "lcars" || d.layout === "standard") CONFIG.layout = d.layout;
  if (d.motion === "potato" || d.motion === "full") CONFIG.motion = d.motion;
  if ([1, 1.1, 1.25, 1.5].indexOf(Number(d.uiScale)) !== -1) CONFIG.uiScale = Number(d.uiScale);
  if (typeof d.operatorPicture === "string" && /^data:image\//.test(d.operatorPicture) && d.operatorPicture.length <= 420000) CONFIG.operatorPicture = d.operatorPicture;
  if (typeof d.operatorBadge === "string" && /^data:image\//.test(d.operatorBadge) && d.operatorBadge.length <= 420000) CONFIG.operatorBadge = d.operatorBadge;
  saveCfg(); applyTheme(); applyMotion(); applyLayout(); applyUiScale(); refreshSeedGreeting();
}
// The badge slot intake (Build 13, issues #150 + #136): ONE
// implementation shared by both Settings rooms — the Operator
// card's slot and the GitHub Account sync card's slot read a
// badge picture file, parse its navi-badge payload, and set
// the SAME single badge state (CONFIG.operatorBadge) the
// account sync consumes. Anything that is not an operator
// badge is refused in plain words, and the badge held on this
// machine — if any — is left exactly as it was.
function badgeSlotFile(file, done) {
  if (!file) { done(false, ""); return; }
  var rd = new FileReader();
  rd.onload = function () {
    var url = String(rd.result || "");
    var payload = badgeParse(url);
    if (!payload) { done(false, "That file is not an Operator Badge — no navi-badge payload with kind \"operator-badge\" inside it. The badge held on this machine is untouched."); return; }
    if (url.length > 420000) { done(false, "That badge picture is too large to keep — the badge state caps at the size a settings backup carries. The badge held on this machine is untouched."); return; }
    CONFIG.operatorBadge = url;
    saveCfg();
    done(true, "Badge slotted — it says " + (payload.name || "(no name)") + (payload.node ? ", minted on node " + payload.node : "") + ". This machine holds one badge state, and the account sync uses this same badge.");
  };
  rd.onerror = function () { done(false, "That file could not be read — the badge held on this machine is untouched."); };
  rd.readAsDataURL(file);
}
function ghBadgeSync() {
  if (CONFIG.backend !== "live") { ghSetMsg("Account sync runs against the live bridge — mock mode simulates backups themselves, not the pull.", false); render(); return; }
  if (!githubState || !githubState.signedIn) { ghSetMsg("Sign in first — the badge points, but the GitHub sign-in is the only key. Nothing was fetched.", true); render(); return; }
  if (badgeSyncBusy) return;
  badgeSyncBusy = true;
  // Build 13 (#136, the field round): the pull's progress and
  // verdict live on the operation card in Chat, so pressing
  // the button goes there AT PRESS TIME — Settings never sits
  // spinning on a fetch whose answer lands in another room.
  // The start carries a 45-second timeout: a bridge that does
  // not answer in time fails in plain words in both rooms and
  // frees the button (the bridge may still finish its pull;
  // if its card lands, it is live in Chat, and nothing ever
  // applies without a confirm on the card).
  ghSetMsg("Pulling the latest backup — its preview and its confirm live on the sync card in Chat.", false);
  goView("Chat");
  var ctl = (typeof AbortController !== "undefined") ? new AbortController() : null;
  var timer = ctl ? setTimeout(function () { ctl.abort(); }, 45000) : null;
  fetch("/api/github/badge-sync/start", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ dashboard: ghDashboardSlice() }), signal: ctl ? ctl.signal : undefined })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (timer) clearTimeout(timer);
      badgeSyncBusy = false;
      var intro = j && j.ok
        ? "Your badge pointed the way and the sign-in did the unlocking — the latest backup's preview is on the card below. Nothing lands until you confirm it there; a thread both machines moved on will arrive whole, as a dated sibling, never merged away."
        : "The badge sync stopped at its first step — the card below has the reason, in plain words.";
      ghSetMsg(j && j.ok ? "The latest backup's preview is on the sync card in Chat — nothing lands until you confirm there." : "The badge sync stopped at its first step — the card in Chat has the reason, in plain words.", !(j && j.ok));
      messages.push({ who: "NAVI", text: intro, exec: null, operation: (j && j.operation) || null });
      persist();
      if (view === "Chat") render();
      if (j && j.operationId) watchOperation(j.operationId);
    })
    .catch(function (err) {
      if (timer) clearTimeout(timer);
      badgeSyncBusy = false;
      var why = (err && err.name === "AbortError")
        ? "The badge sync didn't answer within 45 seconds — the bridge may still be pulling the backup, and if its card lands in Chat it is live there; nothing applies without your confirm. The same note is in Settings, and the button is free to try again."
        : "The bridge didn't answer — the sync never started.";
      ghSetMsg(why, true);
      messages.push({ who: "NAVI", text: why, exec: null, operation: null });
      persist();
      if (view === "Chat") render();
    });
}

/* ------------------- Boot arm/disarm (#159) ------------------- */
// One handler, one state cache, two surfaces: the status
// column's Servers -> Boot row (the home) and Settings ->
// System's Startup card both render from bootStateCache and
// both call bootAction(). After any action the bridge's answer
// carries its fresh re-read, and render() re-fetches /api/status
// on top of that \u2014 the state shown is the system's, never the
// button's hope.
var bootBusy = false, bootMsg = "", bootLingerCmd = "", bootStateCache = null;
var bootFetching = false;
function bootUnitWord(s) { return s === "enabled" ? "enabled" : s === "disabled" ? "not enabled" : "unknown"; }
function bootButtonHtml(id) {
  var armed = !!(bootStateCache && bootStateCache.armed);
  var label = bootBusy ? "Working\u2026" : armed ? "Disarm \u2014 don't start at boot" : "Arm at boot";
  return "<button type='button' id='" + id + "' class='ghost boot-action-btn' data-action='" + (armed ? "disarm" : "arm") + "' style='min-height:44px;margin-top:8px'" + (bootBusy ? " disabled" : "") + ">" + esc(label) + "</button>";
}
function bootMsgHtml() {
  var out = "";
  if (bootMsg) out += "<div class='boot-msg'>" + esc(bootMsg) + "</div>";
  if (bootLingerCmd) out += "<div class='boot-linger-cmd'>Linger still needs one command from a terminal: <span class='mono'>" + esc(bootLingerCmd) + "</span></div>";
  return out;
}
function bootAction() {
  if (bootBusy) return;
  var armed = !!(bootStateCache && bootStateCache.armed);
  bootBusy = true;
  render();
  fetch(armed ? "/api/boot/disarm" : "/api/boot/arm", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      bootStateCache = (j && j.boot) || bootStateCache;
      bootMsg = (j && j.message) || (j && j.ok ? "Done \u2014 the state above is what the system reports now." : "The system did not take it \u2014 the state above is what it reports now.");
      bootLingerCmd = (j && j.lingerCommand) || "";
      bootBusy = false;
      render();
    })
    .catch(function () {
      bootMsg = "The bridge didn't answer \u2014 nothing was changed.";
      bootBusy = false;
      render();
    });
}

/* ------------------- The sync bar (v0.4.10e, #254) ------------------- */
// The head of the status pane, in the seat #250 vacated. Two
// faces, both read from the bridge's sync record on the status
// poll — never from window-local guesses: at rest, "System
// Synced" with the last round's receipt hanging beneath it;
// while a round runs (any trigger — button, app start, DHD
// session, Mesh contact), the phase the bridge reports, its
// segments lit in order, counts only where the bridge
// measures them. A phase that has not advanced in 25 seconds
// names its stall and the honest reason per leg; a failed
// round names its failure on the bar until the next round.
function syncReceiptText(receipt) {
  var c = (receipt && receipt.counts) || {};
  var bits = [];
  if (c.mergedMessages) bits.push(c.mergedMessages + " message" + (c.mergedMessages === 1 ? "" : "s") + " merged into the main thread");
  if (c.foldedGhosts) bits.push(c.foldedGhosts + " ghost thread" + (c.foldedGhosts === 1 ? "" : "s") + " folded in");
  if (c.pairsRepaired) bits.push(c.pairsRepaired + " case/chat pair" + (c.pairsRepaired === 1 ? "" : "s") + " made whole");
  (c.gapTitles || []).forEach(function (t) { bits.push("the chat of “" + t + "” folded away — its case didn't cross"); });
  if (c.tombstones) bits.push(c.tombstones + " deletion" + (c.tombstones === 1 ? "" : "s") + " honored");
  if (c.chatsAdded) bits.push(c.chatsAdded + " chat" + (c.chatsAdded === 1 ? "" : "s") + " arrived");
  // v0.5.13e (#262): shells are named on the bar, not buried —
  // an arrival verified short of its declared substance says
  // so until a fuller copy heals it.
  (c.incomplete || []).forEach(function (t) { bits.push(t + " arrived incomplete — a fuller copy heals it"); });
  // v0.5.1e (#257): the landing names its road and the time it
  // measured — "landed in 3s via Mesh — <peer>", or the repo
  // road in its own words. A receipt from an older build
  // carries neither field and reads exactly as it always has.
  var tail = "";
  if (receipt && (receipt.route || typeof receipt.durationMs === "number")) {
    var via = receipt.route ? (receipt.route.indexOf("Mesh") === 0 ? "via " + receipt.route : "via the backup repo") : "";
    var dur = (typeof receipt.durationMs === "number" && receipt.durationMs >= 0) ? " in " + syncDurText(receipt.durationMs) : "";
    if (via || dur) tail = "landed" + dur + (via ? " " + via : "");
  }
  if (!bits.length) return "Last sync: nothing changed." + (tail ? " " + tail.charAt(0).toUpperCase() + tail.slice(1) + "." : "");
  return "Last sync: " + bits.join(" · ") + (tail ? " · " + tail : "") + ".";
}
function syncDurText(ms) {
  if (ms < 1000) return "<1s";
  return Math.round(ms / 1000) + "s";
}
function renderSyncBar(live) {
  if (CONFIG.backend !== "live") return "";
  var sync = (live && live.sync) || null;
  var prog = (sync && sync.progress) || null;
  var receipt = (sync && sync.receipt) || null;
  var inner;
  if (prog && prog.active) {
    var names = { fetch: "Fetching", merge: "Merging", write: "Writing" };
    var label = names[prog.phase] || "Syncing";
    var since = (typeof prog.phaseAt === "number" && prog.phaseAt) || (typeof prog.startedAt === "number" && prog.startedAt) || 0;
    var secs = since ? Math.max(0, Math.floor(Date.now() / 1000 - since)) : 0;
    var stalled = secs >= 25;
    var order = ["fetch", "merge", "write"];
    var cur = order.indexOf(prog.phase);
    var segs = order.map(function (p, i) { return "<span class='sync-seg" + (i < cur ? " done" : i === cur ? " now" : "") + "'></span>"; }).join("");
    var sub = label;
    if (prog.phase === "write" && prog.counts && prog.counts.filesTotal) sub += " · " + (prog.counts.filesWritten || 0) + " of " + prog.counts.filesTotal + " files";
    if (stalled) {
      var why = prog.phase === "fetch" ? "no answer from the other side yet"
        : prog.phase === "write" ? "the write has not finished"
        : "the merge is taking a while on this machine";
      sub = "Still " + label.toLowerCase() + " after " + secs + "s — " + why + ". It will land or name its failure.";
    }
    inner = "<div class='sync-bar" + (stalled ? " stalled" : "") + "'><span class='dot' style='background:#f5b544'></span><span class='sync-bar-label'>Syncing — " + label + "</span></div>" +
      "<div class='sync-track'>" + segs + "</div><div class='sync-bar-sub'>" + esc(sub) + "</div>";
  } else if (receipt && receipt.ok === false) {
    inner = "<div class='sync-bar failed'><span class='dot' style='background:#f87171'></span><span class='sync-bar-label'>Sync didn't land — " + esc(receipt.message || "the round failed") + "</span></div>";
  } else {
    var ago = sync && sync.lastSyncEpoch ? " · " + syncAgoText(sync.lastSyncEpoch) : "";
    inner = "<div class='sync-bar'><span class='dot' style='background:#34d399'></span><span class='sync-bar-label'>System Synced</span><span class='sync-bar-ago'>" + ago + "</span></div>" +
      (receipt ? "<div class='sync-bar-sub'>" + esc(syncReceiptText(receipt)) + "</div>" : "");
  }
  return "<div class='stat' style='padding-bottom:10px'>" + inner + "</div>";
}

/* ------------------- Boot mode, read live (#142) ------------------- */
function renderBootStatus(boot) {
  if (boot) bootStateCache = boot;
  if (CONFIG.backend !== "live" || !boot) return "";
  // v0.4.10e (#250): the BOOT section follows Settings. When
  // the startup mode Settings holds is not boot mode, the pane
  // does not render the section at all; when it is, the
  // section renders exactly as it always has — its states
  // read from the system, never assumed. Enabling and arming
  // live in Settings → System, as they did.
  if (boot.settingsMode !== "on-boot") return "";
  var unit = function (s) { return s === "enabled" ? "enabled" : s === "disabled" ? "not enabled" : "unknown"; };
  var line = boot.armed ? "Armed — Apex comes up headless at boot" : "Not armed — Apex starts when you open it";
  var dot = boot.armed ? "#34d399" : "#9fb3c8";
  return "<div class='stat'><span class='dot' style='background:" + dot + "'></span><span style='color:var(--dim)'>BOOT</span><br>" +
    "<span class='v mono' style='font-weight:700'>" + esc(line) + "</span>" +
    "<div style='color:var(--dim);font-size:12px;margin-top:2px'>Bridge service " + unit(boot.bridgeUnit) + " · llamafile service " + unit(boot.llamafileUnit) + " · linger " + (boot.linger === true ? "on" : boot.linger === false ? "off" : "unknown") + " — read from this system, never assumed. <b>Server mode</b> in Docs tells the whole story.</div>" + bootButtonHtml("boot-action") + bootMsgHtml() + "</div>";
}

/* ----------------------- Case Files (#137/#138) ----------------------- */
// The first Apex | App. A case is a folder of plain markdown in
// the Vault (cases/<slug>/) — manifest in case.md's frontmatter,
// findings as notes, evidence as files, the timeline one
// newest-first file — so cases ride the Vault's backups and open
// in Obsidian as they lie. The OPEN case is the one Navi files to
// from chat, out loud; the report export (#138) composes from the
// same files and counts what it withholds.
var casesState = null, casesFetching = false;
var caseDetailState = null, caseDetailFetching = false;
// v0.5.9e (#280): the finding whose delete confirm stands
// open in the case view — the finding's file name.
var cfDelArm = "";
var caseMsg = "", caseReportState = null, caseDragOver = false;
var caseEditOpen = false, caseEditMsg = "";
var caseDeleteArm = "";   // Build 15 (#163): the slug whose delete confirm is showing, if any
var caseListDeleteArm = "";   // Build 3 (#177): the same arm, on the cases LIST — the detail card's delete existed since #163, but the list (where cases are actually scanned) had none, which is how the operator came to report case deletion as missing
var caseOpenSlug = "";
try { caseOpenSlug = localStorage.getItem("navi.openCase") || ""; } catch (e) {}
var lastChatExec = null;   // newest terminal/chip output from chat, for filing
function refreshCases() {
  if (CONFIG.backend !== "live" || casesFetching) return;
  casesFetching = true;
  fetch("/api/cases").then(function (r) { return r.json(); }).then(function (j) { casesState = (j && j.cases) || []; casesFetching = false; if (view === "CaseFiles") render(); }).catch(function () { casesFetching = false; });
}
function refreshCaseDetail(slug) {
  if (CONFIG.backend !== "live" || caseDetailFetching) return;
  caseDetailFetching = true;
  fetch("/api/cases/detail?case=" + encodeURIComponent(slug)).then(function (r) { return r.json(); }).then(function (j) { caseDetailFetching = false; if (j && j.ok) { caseDetailState = j.case; if (view === "CaseFiles") render(); } }).catch(function () { caseDetailFetching = false; });
}
// ---------- Threat Intelligence Center (Build 15, #165) ----------
// The pullers, the store, the matcher and the digest live in
// the bridge (intel.py); this view is a window onto their state
// — freshness first, because a stale picture presented as fresh
// is worse than none. Navi carries the same brief in chat.
var intelState = null, intelFetching = false, intelMsg = "";
var intelSelId = "", intelItem = null, intelItemFetching = false;
// v0.5.2e (#259): the catalog search's standing state — one
// search across the Center's CVE and KEV pages alike (the
// catalog holds both; KEV is a flag on the same records).
// Results render as the queue's own rows and open in the
// record region through the same data-ti-sel door.
var intelSearch = null, intelSearchBusy = false;
function intelPost(path, body, cb) { casePost(path, body, cb); }
function refreshIntel() {
  if (intelFetching) return;
  intelFetching = true;
  fetch("/api/intel").then(function (r) { return r.json(); }).then(function (j) { intelState = j; intelFetching = false; if (view === "ThreatIntel") render(); }).catch(function () { intelFetching = false; });
}
function refreshIntelItem(cve) {
  intelItemFetching = true;
  fetch("/api/intel/item?cve=" + encodeURIComponent(cve)).then(function (r) { return r.json(); }).then(function (j) { intelItemFetching = false; if (j && j.ok) intelItem = j.item; if (view === "ThreatIntel") render(); }).catch(function () { intelItemFetching = false; });
}
function intelSev(cvss) {
  if (cvss === null || cvss === undefined) return "info";
  return cvss >= 9 ? "critical" : cvss >= 7 ? "high" : cvss >= 4 ? "medium" : "low";
}
function intelBandColor(band) { return { fresh: "#34d399", aging: "#f5b544", stale: "#f87171", never: "var(--dim)" }[band] || "var(--dim)"; }
function intelItemHtml(it, sel) {
  var badges = "";
  if (it.kev) badges += " <span class='mono' style='font-size:11px;color:#f87171'>KEV" + (it.kevDue ? " · due " + esc(it.kevDue) : "") + "</span>";
  if (it.exploits && it.exploits.length) badges += " <span class='mono' style='font-size:11px;color:#f5b544'>EXPLOIT · EDB-" + esc(it.exploits[0].edb) + ((it.exploitCount || it.exploits.length) > 1 ? " +" + ((it.exploitCount || it.exploits.length) - 1) : "") + "</span>";
  if (it.matched) badges += " <span class='mono' style='font-size:11px;color:#34d399'>" + (it.matched.grade === "exact" ? "MATCHES" : "mentions") + " " + esc(it.matched.via) + ": " + esc(it.matched.term) + "</span>";
  var meta = "";
  if (it.cvss !== null && it.cvss !== undefined) meta += "CVSS " + it.cvss + " ";
  if (it.epss !== null && it.epss !== undefined) meta += "· EPSS " + Math.round(it.epss * 100) + "% ";
  if (it.published) meta += "· " + esc(String(it.published).slice(0, 10));
  return "<button class='ghost tic-qitem' data-ti-sel='" + esc(it.id) + "'" + (sel ? " style='border-color:var(--cy)'" : "") + "><b style='color:" + caseSevColor(intelSev(it.cvss)) + "'>" + esc(it.id) + "</b>" + badges + "<br><span style='font-size:13px'>" + esc(it.title || "") + "</span><br><span class='mono' style='color:var(--dim);font-size:11px'>" + esc(meta) + "</span></button>";
}
// ---------- Awareness home (0.2.0e Build 7: #188 / #189 / Add. 4 / Add. 5) ----------
// The Threat Intelligence Center is the awareness home. Its
// landing is a Dashboard — one glance card per family, each
// with its freshness stamp, each opening its full view — and
// every family gets its own tab carrying as much data as the
// source gives: News (the wire), CVEs (the Center's records),
// CTFs (the board), Bounties, and the Center itself. Articles
// open in the in-app reader where extraction works; where it
// cannot, the fallback is named, never faked.
var intelTab = "home";
// The selected Wire tab survives reload (v0.4.1e, #230):
// every operator-driven tab change routes through
// setIntelTab, which persists it; boot restores it here.
// Retired keys (dashboard, cves) still normalize inside
// threatIntelHtml, as before.
try { intelTab = localStorage.getItem("navi.intelTab") || "home"; } catch (e) {}
function setIntelTab(t) {
  intelTab = t;
  try { localStorage.setItem("navi.intelTab", t); } catch (e) {}
}
var wireState = null, wireFetching = false, wireMsg = "";
// The living greeting (v0.5.13e, #288): the bridge composes
// it from her actual statuses; Home's Brief asks once per
// visit and speaks its answer — headline, quip, then the
// brief, with the Pulse digest folded in. Until it answers,
// the Brief composes as it always has.
var greetingState = null, greetingFetching = false;
function loadGreeting() {
  if (greetingFetching) return;
  greetingFetching = true;
  fetch("/api/greeting").then(function (r) { return r.json(); }).then(function (j) { greetingState = j; greetingFetching = false; if (view === "Home") render(); }).catch(function () { greetingFetching = false; });
}
var ctfState = null, ctfFetching = false, ctfMsg = "";
var bountyState = null, bountyFetching = false, bountyMsg = "";
var wireReader = null, wireReaderBusy = false;
// Pulse (v0.5.13e, #287): the tab's store mirror — the bridge
// owns the watches; this page holds the last answer and asks
// again after every act, the house pattern.
var pulseState = null, pulseFetching = false, pulseMsg = "";
function refreshPulse() {
  if (pulseFetching) return;
  pulseFetching = true;
  fetch("/api/pulse").then(function (r) { return r.json(); }).then(function (j) { pulseState = j; pulseFetching = false; if (view === "ThreatIntel") render(); }).catch(function () { pulseFetching = false; });
}
function wirePulsesHtml() {
  var h = "";
  if (pulseMsg) h += "<p style='font-size:13px'>" + esc(pulseMsg) + "</p>";
  if (!pulseState) { if (!pulseFetching) refreshPulse(); return h + "<div class='card'>Gathering the pulses\u2026</div>"; }
  var s = pulseState;
  h += "<div class='card'><b>Pulses — the topics Navi keeps watch on</b>" +
    "<p style='font-size:13px;margin-top:6px'>Navi can only keep a pulse where she can read. These topics are watched across the news networks enabled in the News tab — enable more there, or add your own, and her reach grows.</p>" +
    "<p style='font-size:12px;color:var(--dim)'>Reading now: " + ((s.networks || []).length ? (s.networks || []).map(function (n) { return esc(n); }).join(" · ") : "no networks enabled — the wire is off, or every source is disabled in the News tab") + ". She checks every few hours, and again as she composes her greeting; something new lands as one batched notice per watch, never one per article. Or say it in chat: <span class='mono'>keep a pulse on Epic MyChart</span>.</p>" +
    "<div class='row' style='margin-top:10px;gap:8px;flex-wrap:wrap'><input id='pulse-topic' type='text' placeholder='A topic to watch — a system, a vendor, a threat' style='flex:1;min-width:200px;min-height:40px' aria-label='Topic to watch'><button class='act' id='pulse-add' style='min-height:40px'>Keep a pulse on it</button><button class='ghost' id='pulse-run' style='min-height:40px'>Check now</button></div></div>";
  var watches = s.watches || [];
  if (!watches.length) {
    h += "<div class='card' style='margin-top:12px'><p style='font-size:13px;color:var(--dim)'>No watches yet. Flag a topic above, or tell Navi in chat — the list, the notices and the greeting all draw from the same store.</p></div>";
    return h;
  }
  watches.forEach(function (w) {
    h += "<div class='card' style='margin-top:12px'><div style='display:flex;flex-wrap:wrap;gap:8px;align-items:center;justify-content:space-between'><b>" + esc(w.topic) + "</b><span class='mono' style='font-size:11px;color:var(--dim)'>" + (w.state === "paused" ? "PAUSED" : "WATCHING") + "</span></div>";
    h += "<p style='font-size:12px;color:var(--dim);margin-top:4px'>Last checked " + (w.lastCheckIso ? esc(w.lastCheckIso.replace("T", " ").replace("Z", " UTC")) : "not yet") + (w.newCount ? " · <b style='color:var(--tx)'>" + w.newCount + " new</b>" : "") + (w.case ? " · feeding the case <b>" + esc(w.case) + "</b> as reference" : "") + "</p>";
    if (w.coverage === "none") h += "<p style='font-size:12px;color:#f5c542'>No enabled network coverage — enable the wire or add a network in the News tab and this watch starts reading.</p>";
    (w.hits || []).forEach(function (hit) {
      h += "<div style='margin-top:6px'><a href='" + esc(hit.link || "#") + "' target='_blank' rel='noopener' style='color:var(--cy)'>" + esc(hit.title || "") + "</a> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(hit.sourceLabel || hit.source || "") + "</span></div>";
    });
    h += "<div class='row' style='margin-top:8px;gap:8px;flex-wrap:wrap'>";
    if (w.state === "paused") h += "<button class='ghost' data-pulse-resume='" + esc(w.id) + "' style='min-height:40px'>Resume</button>";
    else h += "<button class='ghost' data-pulse-pause='" + esc(w.id) + "' style='min-height:40px'>Pause</button>";
    h += "<button class='ghost' data-pulse-remove='" + esc(w.id) + "' style='min-height:40px'>Remove</button>";
    if ((s.cases || []).length) {
      h += "<select data-pulse-case='" + esc(w.id) + "' style='min-height:40px' aria-label='Feed a case'>" + "<option value=''>Feed a case\u2026</option>" + (s.cases || []).map(function (c) { return "<option value='" + esc(c.slug) + "'" + (w.case === c.slug ? " selected" : "") + ">" + esc(c.name) + "</option>"; }).join("") + "</select>";
      h += "<button class='ghost' data-pulse-attach='" + esc(w.id) + "' style='min-height:40px'>Attach</button>";
    }
    h += "</div></div>";
  });
  return h;
}
var newCaseState = null; // v0.4.5e (#241): the New-case-from-chat form's state, while it stands
function refreshWire() {
  if (wireFetching) return;
  wireFetching = true;
  fetch("/api/wire").then(function (r) { return r.json(); }).then(function (j) { wireState = j; wireFetching = false; if (view === "ThreatIntel" || view === "CTFBoard") render(); }).catch(function () { wireFetching = false; });
}
function refreshCtf() {
  if (ctfFetching) return;
  ctfFetching = true;
  fetch("/api/ctf").then(function (r) { return r.json(); }).then(function (j) { ctfState = j; ctfFetching = false; if (view === "ThreatIntel" || view === "CTFBoard") render(); }).catch(function () { ctfFetching = false; });
}
function refreshBounties() {
  if (bountyFetching) return;
  bountyFetching = true;
  fetch("/api/bounties").then(function (r) { return r.json(); }).then(function (j) { bountyState = j; bountyFetching = false; if (view === "ThreatIntel" || view === "CTFBoard") render(); }).catch(function () { bountyFetching = false; });
}
function ticFresh(ts) {
  if (!ts) return "never fetched";
  var ageMin = Math.max(0, Math.round((Date.now() / 1000 - ts) / 60));
  if (ageMin < 1) return "fetched under a minute ago";
  if (ageMin < 90) return "fetched " + ageMin + " min ago";
  return "fetched " + Math.round(ageMin / 6) / 10 + " h ago";
}
function ticLocal(ts) {
  if (!ts) return "";
  try { return new Date(ts * 1000).toLocaleString(); } catch (e) { return ""; }
}
// Build 3 (#210) fused the Dashboard and the Center into this
// surface and re-laned the wire's families: News carries news
// only, the wire's CVE items get their own family, its KEV
// items likewise; the Center's graded queue is its own family.
//
// Build 4 (#213): Home is the BOARD only — the Situation
// strip and the four glance cards, nothing else. The feed-
// health pill field moved to the top of the News family; the
// tab strip is gone — each family is its own view, reached
// through the cards' Open buttons and the nav. Home itself is
// built as a HOST surface: the board is its first resident,
// not its permanent definition (Alex's addendum on #213 —
// the homepage "will be a lot of things"), so residents are
// composed in homeSurfaceHtml and later ones append there
// without another surgery. Board-only is today's content
// ruling, never an architectural cap.
var TIC_FAMILIES = ["news", "cve", "kev", "ctfs", "pulses", "bounties", "center"];
function threatIntelHtml() {
  // Retired keys land somewhere honest: the old Dashboard key
  // is the board itself now; the old CVEs scope folded into
  // the Center family.
  if (intelTab === "dashboard") setIntelTab("home");
  if (intelTab === "cves") setIntelTab("center");
  if (CONFIG.backend !== "live") {
    return "<h3>Home — The Wire</h3><div class='card'>The awareness home works through the live bridge — the stores live in this install's config. Switch the backend to Live in Settings and it will be here.</div>";
  }
  if (TIC_FAMILIES.indexOf(intelTab) !== -1) {
    // A family view: the family's own content, with one way
    // back to the board. In the CTFBoard view the family IS
    // the view (the nav is the way out), so no back row there.
    var fh = "";
    if (view === "ThreatIntel") fh += "<div class='row' style='margin-bottom:10px'><button class='ghost' data-tic-go='home' style='min-height:40px'>\u2190 Home — the board</button></div>";
    // The family pages carry the Wire's tab bar (#230): the
    // four families switch in place, inside The Wire — no
    // bounce to Home or the Center to read a family's list.
    // In the CTFBoard view the bar rides without the back row.
    if (["news", "cve", "kev", "ctfs", "pulses"].indexOf(intelTab) !== -1) fh += wireTabsHtml();
    if (wirePullMsg) fh += "<p style='font-size:13px'>" + esc(wirePullMsg) + "</p>";
    if (intelTab === "cve") fh += wireKindHtml("cve");
    else if (intelTab === "kev") fh += wireKindHtml("kev");
    else if (intelTab === "ctfs") fh += ctfTabHtml();
    else if (intelTab === "pulses") fh += wirePulsesHtml();
    else if (intelTab === "bounties") fh += bountiesTabHtml();
    else if (intelTab === "center") fh += ticCenterHtml("center");
    else fh += wireTabHtml();
    if (wireReader) fh += wireReaderHtml();
    if (wirePullPick) fh += wirePullPickerHtml();
    return fh;
  }
  setIntelTab("home");
  var h = "<h3>Home — The Wire</h3>";
  // v0.4.7e (#243): the self-explaining intro paragraph is retired — Home leads with Navi's Brief now, not with a description of itself.
  h += homeSurfaceHtml();
  if (wireReader) h += wireReaderHtml();
  return h;
}
function homeBriefHtml() {
  // Navi's Brief (v0.4.1e, #230): Home's second resident —
  // her summary of the picture, composed ONLY from data
  // this page already holds: the wire's store, the Center's
  // counts, the CTF board's lists, the Mesh state the frame
  // polls, her model's state, and whether a call is live.
  // No new pipelines, no assistant machinery: when a store
  // has not answered yet, its line is simply absent rather
  // than invented — and if nothing has answered, the brief
  // says she is still gathering.
  var hour = new Date().getHours();
  var greet = hour < 5 ? "Still up" : hour < 12 ? "Morning" : hour < 18 ? "Afternoon" : "Evening";
  var lines = [];
  if (wireState && wireState.items) {
    var n = 0, c = 0, k = 0;
    wireState.items.forEach(function (it) { if (it.kind === "news") n++; else if (it.kind === "cve") c++; else if (it.kind === "kev") k++; });
    var dead = (wireState.sources || []).filter(function (x) { return !!x.error; }).length;
    lines.push("The wire holds " + n + (n === 1 ? " headline" : " headlines") + ", " + c + " new CVEs and " + k + " KEV additions" + (dead ? " — and " + dead + (dead === 1 ? " feed is" : " feeds are") + " not answering; the News page names " + (dead === 1 ? "it" : "them") : " — every feed answering") + ".");
  }
  if (intelState && intelState.kevTotal !== undefined) {
    lines.push("The Center's catalog holds " + (intelState.kevTotal || 0) + " known-exploited vulnerabilities; " + (intelState.matchedCount || 0) + " matched to you, with " + ((intelState.queue || []).length) + " in the queue.");
  }
  if (ctfState && ctfState.ok !== false) {
    var run = ctfState.running || [], up = ctfState.upcoming || [];
    if (run.length) lines.push(run.length + (run.length === 1 ? " CTF is" : " CTFs are") + " running now — " + run[0].title + (run.length > 1 ? " and " + (run.length - 1) + " more" : "") + ".");
    else if (up.length) lines.push("Next on the CTF board: " + up[0].title + ", starting " + ticLocal(up[0].start) + ".");
  }
  if (meshState) {
    if (meshState.enabled) {
      var pn = (meshState.peers || []).length;
      lines.push("The Mesh is on — " + pn + (pn === 1 ? " machine" : " machines") + " paired with this one.");
    } else {
      lines.push("The Mesh is off — she is answering from this machine alone.");
    }
  }
  if (callState && callState !== "idle") lines.push("You are on a call with her right now — she is " + callState + ".");
  lines.push(llama && llama.status === "running" ? "Her model is up — she is here." : "Her model is not running — she answers in command-line mode until it is back.");
  // v0.4.7e (#243): the Brief as a designed element — her
  // portrait, a caps label, the greeting set as a headline,
  // her first line as the lead and the rest quiet beneath it.
  if (!greetingState && !greetingFetching) loadGreeting();
  var briefHead = greet + ".";
  var briefLines = lines;
  if (greetingState && greetingState.line) {
    briefHead = greetingState.line + ".";
    briefLines = [greetingState.quip].concat(greetingState.brief || [], lines.filter(function (l) { return l.indexOf("The wire holds ") !== 0 && l.indexOf("The Mesh is ") !== 0; }));
  }
  var h = "<div class='card brief-card' id='tic-brief'><div class='brief-head'><img class='presence-av brief-av' src='assets/navi-portrait.jpg' alt='Navi'><div><div class='brief-label'>NAVI'S BRIEF</div><div class='brief-greet'>" + esc(briefHead) + "</div></div></div>";
  if (!briefLines.length) h += "<div class='brief-line brief-lead'>Still gathering \u2014 the wire has not answered yet. The board below fills in as it does.</div>";
  else h += briefLines.map(function (l, li) { return "<div class='brief-line" + (li === 0 ? " brief-lead" : "") + "'>" + esc(l) + "</div>"; }).join("");
  return h + "</div>";
}
function homeSurfaceHtml() {
  // The residents of Home, in order. The board was the first
  // (#213); Navi's Brief is the second (v0.4.1e, #230) and
  // rides ahead of it — her read of the picture first, the
  // rows below. Later residents append here — the surface is
  // a host, so nothing below this line needs to change when
  // Home grows.
  var h = "";
  // v0.4.7e (#242): Needs your eyes rides FIRST — ahead of
  // the Brief — whenever anything is unread; it returns ""
  // when nothing is, so a quiet Home is unchanged.
  h += homeEyesHtml();
  h += homeBriefHtml();
  h += homeDayHtml();
  h += homeBoardHtml();
  return h;
}

// --- Brief v2 (v0.6.2e, #304) -----------------------------------
// The brief is a markdown document; no surface presents the
// markdown. This parser is the page's half of the one grammar
// the bridge composes in: sections, item bullets with their
// leading time and trailing [state · pushed ×N] marks, GAP
// bullets, and "Navi: " bullets folding into the row above.
function briefParseMd(md) {
  var doc = { day: "", sections: [] };
  var cur = null;
  String(md || "").split("\n").forEach(function (raw) {
    var line = raw.replace(/\s+$/, "");
    if (line.indexOf("# ") === 0) { doc.day = line.slice(2).replace("Day brief — ", "").trim(); return; }
    if (line.indexOf("## ") === 0) { cur = { title: line.slice(3).trim(), rows: [] }; doc.sections.push(cur); return; }
    if (line.indexOf("- ") !== 0 || !cur) return;
    var body = line.slice(2).trim();
    if (body.indexOf("Navi: ") === 0) {
      if (cur.rows.length) cur.rows[cur.rows.length - 1].quip = body.slice(6).trim();
      return;
    }
    var row = { text: body, time: "", state: "", count: 0, quip: "", gap: false };
    if (body.indexOf("GAP — ") === 0) { row.gap = true; row.text = body.slice(6).trim(); cur.rows.push(row); return; }
    var tm = /^(\d{1,2}:\d{2})\s—\s/.exec(body);
    if (tm) { row.time = tm[1]; body = body.slice(tm[0].length); row.text = body; }
    var mk = /\s\[([^\]]+)\]\s*$/.exec(body);
    if (mk) {
      row.text = body.slice(0, mk.index).replace(/\s+$/, "");
      mk[1].split(" · ").forEach(function (part) {
        part = part.trim();
        if (part.indexOf("pushed ×") === 0) { var n = parseInt(part.slice(8), 10); if (!isNaN(n)) row.count = n; }
        else if (part) row.state = part;
      });
    }
    cur.rows.push(row);
  });
  return doc;
}
function briefListHtml(md, inChat) {
  var doc = briefParseMd(md);
  if (!doc.sections.length) return "";
  var h = "<div class='brief-list'>";
  doc.sections.forEach(function (sec) {
    h += "<div class='brief-sec'>" + esc(sec.title.toUpperCase()) + "</div>";
    sec.rows.forEach(function (r) {
      h += "<div class='brief-row" + (r.gap ? " gap" : "") + "'>" +
        (r.time ? "<span class='b-time'>" + esc(r.time) + "</span>" : "") +
        "<span>" + esc(r.text) + "</span>" +
        (r.state ? "<span class='b-state'>" + esc(r.state.toUpperCase()) + "</span>" : "") +
        (r.count > 1 && r.state.indexOf("pushed") < 0 ? "<span class='b-count'>PUSHED ×" + r.count + "</span>" : "") + "</div>";
      if (r.quip) h += "<div class='brief-quip'><span class='b-who'>NAVI</span><span>" + esc(r.quip) + "</span></div>";
    });
  });
  return h + "</div>";
}
function isBriefDoc(text) {
  return typeof text === "string" && text.indexOf("# Day brief — ") === 0;
}

// --- Her working day (v0.6.0e, #294/#295/#296) -------------------
// Home's third resident: the day book the bridge holds — the
// brief as a stored record, the standing reminders (fired
// ones first, each with its Acknowledge), the standing
// orders with their last run, and a plain capture line. The
// page is a view; every act goes through the bridge's
// deterministic door and the answer it returns IS the state.
var dayState = null, dayFetching = false, dayMsg = "";
function loadDay() {
  if (dayFetching) return;
  dayFetching = true;
  fetch("/api/day").then(function (r) { return r.json(); }).then(function (j) { dayState = (j && j.day) || j; dayFetching = false; if (view === "Home") render(); }).catch(function () { dayFetching = false; });
}
function dayWhen(ts) { if (!ts) return ""; try { return new Date(ts * 1000).toLocaleString(); } catch (e) { return ""; } }
function dayAct(body) {
  fetch("/api/day", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.day) dayState = j.day;
    dayMsg = (j && (j.reply || j.error)) || "";
    render();
  }).catch(function () { dayMsg = "The bridge did not answer — nothing was recorded."; render(); });
}
function homeDayHtml() {
  if (!dayState && !dayFetching) loadDay();
  var h = "<div class='card' id='tic-day'><div class='brief-label'>HER WORKING DAY</div>";
  if (dayMsg) h += "<div class='brief-line' style='margin-top:6px'>" + esc(dayMsg) + "</div>";
  if (!dayState) return h + "<div class='brief-line brief-lead' style='margin-top:6px'>Gathering her day\u2026</div></div>";
  var d = dayState;
  var brief = d.brief;
  if (brief && (brief.md || brief.text)) {
    // v0.6.2e (#304): the card draws the list from the stored
    // document — never the markdown itself as the face.
    h += "<div style='margin-top:8px'><b>Today's brief</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(brief.day || "") + "</span></div>";
    h += briefListHtml(brief.md || brief.text, false);
  } else if (brief && brief.lines && brief.lines.length) {
    h += "<div style='margin-top:8px'><b>Today's brief</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(brief.day || "") + "</span></div>";
    brief.lines.forEach(function (line) { h += "<div class='brief-line'>" + esc(line) + "</div>"; });
  } else {
    h += "<div class='brief-line' style='margin-top:6px'>No brief composed yet today.</div>";
  }
  h += "<div class='row' style='margin-top:8px'><button type='button' class='ghost' data-day-brief='1' style='min-height:44px'>Compose the brief</button></div>";
  var fired = (d.reminders || []).filter(function (r) { return r.state === "fired"; });
  var standing = (d.reminders || []).filter(function (r) { return r.state === "standing"; });
  if (fired.length || standing.length) {
    h += "<div style='margin-top:10px'><b>Reminders</b></div>";
    fired.forEach(function (r) {
      h += "<div class='eyes-row'><div class='eyes-main'><b>" + esc(r.text || "") + "</b><div class='eyes-body'>Due " + esc(dayWhen(r.dueAt)) + " \u2014 it fired and waits for your word." + ((r.deferrals || 0) > 1 ? " Pushed \u00d7" + r.deferrals + "." : "") + "</div></div><button type='button' class='ghost' data-day-postpone='" + esc(r.id) + "' style='min-height:44px'>Push to tomorrow</button><button type='button' class='ghost' data-day-ack='" + esc(r.id) + "' style='min-height:44px'>Acknowledge</button></div>";
    });
    standing.forEach(function (r) {
      h += "<div class='brief-line'>" + esc(r.text || "") + " <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(dayWhen(r.dueAt)) + "</span></div>";
    });
  }
  if ((d.orders || []).length) {
    h += "<div style='margin-top:10px'><b>Standing orders</b></div>";
    d.orders.forEach(function (o) {
      var last = (o.executions && o.executions.length) ? o.executions[o.executions.length - 1] : null;
      h += "<div class='eyes-row'><div class='eyes-main'><b>" + esc(o.text || "") + "</b><div class='eyes-body'>" + esc(o.verb || "") + " \u00b7 " + esc(o.trigger ? (o.trigger.kind === "daily" ? "daily " + dayWhen(o.trigger.at) : o.trigger.kind) : "") + (last ? " \u00b7 last: " + esc(last.outcome || "") : "") + "</div></div><button type='button' class='ghost' data-day-revoke='" + esc(o.id) + "' style='min-height:44px'>Revoke</button></div>";
    });
  }
  h += "<div class='row' style='margin-top:10px'><input id='day-cap-text' type='text' placeholder='Note something for her day\u2026' style='flex:1;min-height:44px'><button type='button' class='act' data-day-capture='1' style='min-height:44px'>Keep it</button></div>";
  h += "<div class='eyes-body' style='margin-top:6px'>Kept in her day book \u2014 the same book every seat reads. Ask her in chat to recall it, or to remind you.</div>";
  return h + "</div>";
}
function wireDay() {
  document.querySelectorAll("[data-day-ack]").forEach(function (b) { b.onclick = function () { dayAct({ action: "ack", id: b.getAttribute("data-day-ack") }); }; });
  document.querySelectorAll("[data-day-postpone]").forEach(function (b) { b.onclick = function () { dayAct({ action: "postpone", id: b.getAttribute("data-day-postpone"), when: "tomorrow" }); }; });
  document.querySelectorAll("[data-day-revoke]").forEach(function (b) { b.onclick = function () { dayAct({ action: "revoke", id: b.getAttribute("data-day-revoke") }); }; });
  document.querySelectorAll("[data-day-brief]").forEach(function (b) { b.onclick = function () { dayAct({ action: "brief" }); }; });
  document.querySelectorAll("[data-day-capture]").forEach(function (b) { b.onclick = function () { var inp = document.getElementById("day-cap-text"); var t = inp ? inp.value : ""; if (t && t.trim()) dayAct({ action: "capture", text: t }); }; });
}


function homeBoardHtml() {
  // The board (#213): the Situation strip and the four glance
  // cards — the fusion of Build 3, kept whole. The glance
  // cards' Open buttons are the doors to the family views.
  if (!wireState && !wireFetching) refreshWire();
  if (!intelState && !intelFetching) refreshIntel();
  // v0.4.7e (#243): the Situation strip becomes glanceable
  // tiles — a figure over a caps micro-label — built ONLY
  // from data this page already holds (the Center's counts,
  // the wire's counts and feed health, the Mesh state the
  // frame polls). The two prose paragraphs go; the wire's
  // staleness sentence stays as one quiet stamp line.
  var tiles = [];
  function sitTile(fig, label, figStyle) { tiles.push("<div class='sit-tile'><div class='sit-fig'" + (figStyle ? " style='" + figStyle + "'" : "") + ">" + fig + "</div><div class='sit-label'>" + label + "</div></div>"); }
  if (intelState) {
    sitTile(esc(intelState.kevTotal || 0), "KEV catalog");
    sitTile(esc(intelState.matchedCount || 0), "Matched to you", (intelState.matchedCount || 0) > 0 ? "color:var(--cy)" : "");
    sitTile(esc((intelState.queue || []).length), "In the queue");
  }
  if (wireState && wireState.items) {
    var wn = 0, wc = 0, wk = 0;
    wireState.items.forEach(function (it) { if (it.kind === "news") wn++; else if (it.kind === "cve") wc++; else if (it.kind === "kev") wk++; });
    sitTile(wn, "Headlines");
    sitTile(wc, "New CVEs");
    sitTile(wk, "KEV additions");
    var srcs = wireState.sources || [];
    var down = srcs.filter(function (x) { return !!x.error; }).length;
    sitTile((srcs.length - down) + "/" + srcs.length, "Feeds answering", down > 0 ? "color:var(--red)" : "");
  }
  if (meshState) {
    var pn2 = (meshState.peers || []).length;
    sitTile(meshState.enabled ? "On" : "Off", "Mesh \u00b7 " + pn2 + (pn2 === 1 ? " peer" : " peers"));
  }
  var h = "<div class='card' id='tic-home'><div class='sit-label sit-head'>SITUATION</div>";
  if (tiles.length) h += "<div class='sit-grid'>" + tiles.join("") + "</div>";
  else h += "<div class='sit-stamp'>Gathering the picture\u2026</div>";
  if (wireState && wireState.staleness) h += "<div class='sit-stamp mono'>" + esc(wireState.staleness) + "</div>";
  h += "</div>";
  h += ticDashboardHtml();
  return h;
}
var wirePillsOpen = true;
try { wirePillsOpen = localStorage.getItem("navi.wirePills") !== "closed"; } catch (e) {}
function wirePillsHtml() {
  // Feed health (Build 3, #210): the source pills live in a
  // collapsible field. Collapsed, they cost one slim row that
  // still NAMES the count of feeds not answering; expanded,
  // the full pill set with each failure's own sentence. The
  // state persists per operator (this browser's storage, the
  // same place the chat latch persists).
  var s = wireState;
  if (!s) return "";
  var srcs = s.sources || [];
  var dead = srcs.filter(function (x) { return !!x.error; }).length;
  var h = "<div class='card' style='margin-top:12px'><button class='ghost' id='wire-pills-toggle' aria-expanded='" + (wirePillsOpen ? "true" : "false") + "' aria-label='Toggle the feed health pills' style='min-height:44px;width:100%;text-align:left'><b>Feed health</b> \u00b7 " + srcs.length + " sources" + (dead ? " \u00b7 <span style='color:#f87171'>" + dead + " not answering</span>" : " \u00b7 all answering") + "<span style='float:right'>" + (wirePillsOpen ? "\u25be" : "\u25b8") + "</span></button>";
  h += "<div class='wire-pills-body'" + (wirePillsOpen ? "" : " style='display:none'") + ">";
  h += "<div class='tic-fresh' style='margin-top:8px'>";
  srcs.forEach(function (src) {
    h += "<span class='tic-chip'><span style='color:" + (src.error ? "#f87171" : "#34d399") + "'>\u25cf</span>&nbsp;" + esc(src.label) + (src.custom ? " (yours)" : "") + " \u00b7 " + (src.error ? "not answering" : ticFresh(src.lastOk)) + "</span>";
  });
  h += "</div>";
  srcs.forEach(function (src) { if (src.error) h += "<p style='color:#f87171;font-size:12px;margin-top:6px'>" + esc(src.error) + "</p>"; });
  h += "</div></div>";
  return h;
}
function ticDashboardHtml() {
  var h = "<div style='display:grid;grid-template-columns:repeat(auto-fit,minmax(270px,1fr));gap:12px'>";
  // -- Top news (the wire)
  h += "<div class='card'><h3 style='margin-top:0'>Top news</h3>";
  if (!wireState) { if (!wireFetching) refreshWire(); h += "<p style='color:var(--dim)'>Gathering the wire…</p>"; }
  else if (wireState.ok === false) h += "<p style='color:#f87171'>" + esc(wireState.error || "The wire is unavailable.") + "</p>";
  else {
    h += "<p class='mono' style='font-size:11px;color:var(--dim)'>" + esc(wireState.staleness || "") + "</p>";
    var news = (wireState.items || []).filter(function (it) { return it.kind === "news"; }).slice(0, 4);
    if (!news.length) h += "<p style='color:var(--dim)'>No headlines yet — the first fetch brings them.</p>";
    news.forEach(function (it) { h += "<div style='margin-top:6px'><button class='ghost' data-wire-read='" + esc(encodeURIComponent(it.link)) + "' style='text-align:left;min-height:40px'>" + esc(it.title) + "</button><div class='mono' style='font-size:11px;color:var(--dim)'>" + esc(it.sourceLabel || "") + (it.published ? " · " + esc(ticLocal(it.published)) : "") + "</div></div>"; });
  }
  h += "<div class='row' style='margin-top:10px'><button class='act' data-tic-go='news' style='min-height:40px'>Open the wire</button></div></div>";
  // -- Newest CVEs (the Center)
  h += "<div class='card'><h3 style='margin-top:0'>Newest CVEs</h3>";
  if (!intelState) { if (!intelFetching) refreshIntel(); h += "<p style='color:var(--dim)'>Gathering the picture…</p>"; }
  else if (intelState.ok === false) h += "<p style='color:#f87171'>" + esc(intelState.error || "The Center is unavailable.") + "</p>";
  else {
    h += "<p class='mono' style='font-size:11px;color:var(--dim)'>" + esc(intelState.kevTotal || 0) + " known-exploited · " + esc(intelState.matchedCount || 0) + " matched to you</p>";
    var q = (intelState.queue || []).slice(0, 4);
    if (!q.length) h += "<p style='color:var(--dim)'>The queue is empty — pull from the CVEs tab, or let the cadence work.</p>";
    q.forEach(function (it) { h += "<div style='margin-top:6px'><b style='color:" + caseSevColor(intelSev(it.cvss)) + "'>" + esc(it.id) + "</b> <span style='font-size:13px'>" + esc(it.title || "") + "</span></div>"; });
  }
  h += "<div class='row' style='margin-top:10px'><button class='act' data-tic-go='center' style='min-height:40px'>Open the Center</button></div></div>";
  // -- Next CTFs (the board)
  h += "<div class='card'><h3 style='margin-top:0'>Next CTFs</h3>";
  if (!ctfState) { if (!ctfFetching) refreshCtf(); h += "<p style='color:var(--dim)'>Gathering the board…</p>"; }
  else if (ctfState.ok === false) h += "<p style='color:#f87171'>" + esc(ctfState.error || "The board is unavailable.") + "</p>";
  else {
    h += "<p class='mono' style='font-size:11px;color:var(--dim)'>" + esc(ctfState.staleness || "") + "</p>";
    var evs = (ctfState.running || []).concat(ctfState.upcoming || []).slice(0, 3);
    if (!evs.length) h += "<p style='color:var(--dim)'>Nothing on the board in the window.</p>";
    evs.forEach(function (e, i) { h += "<div style='margin-top:6px'><button type='button' class='ghost tic-ev-read' data-ctf-read='" + i + "' aria-label='Read this event in the reader' style='text-align:left;min-height:40px'><b>" + esc(e.title) + "</b></button><div class='mono' style='font-size:11px;color:var(--dim)'>" + (e.start * 1000 <= Date.now() ? "running now · ends " + esc(ticLocal(e.finish)) : "starts " + esc(ticLocal(e.start))) + " · " + esc(e.format || "") + "</div></div>"; });
  }
  h += "<div class='row' style='margin-top:10px'><button class='act' data-tic-go='ctfs' style='min-height:40px'>Open the board</button></div></div>";
  // -- Bounties
  h += "<div class='card'><h3 style='margin-top:0'>Bounties</h3>";
  if (!bountyState) { if (!bountyFetching) refreshBounties(); h += "<p style='color:var(--dim)'>Gathering the programs…</p>"; }
  else if (bountyState.ok === false) h += "<p style='color:#f87171'>" + esc(bountyState.error || "The bounty board is unavailable.") + "</p>";
  else {
    h += "<p class='mono' style='font-size:11px;color:var(--dim)'>" + esc(bountyState.staleness || "") + "</p>";
    h += "<p style='font-size:13px'>" + esc((bountyState.programs || []).length) + " public programs · " + esc(bountyState.payingCount || 0) + " bounty-paying (ProjectDiscovery Chaos).</p>";
    (bountyState.programs || []).filter(function (p) { return p.isNew; }).slice(0, 3).forEach(function (p) { h += "<div style='margin-top:4px'><span class='mono' style='font-size:11px;color:#34d399'>NEW</span> <b>" + esc(p.name) + "</b> <span style='color:var(--dim);font-size:12px'>" + esc(p.platform || "") + "</span></div>"; });
  }
  h += "<div class='row' style='margin-top:10px'><button class='act' data-tic-go='bounties' style='min-height:40px'>Open the bounties</button></div></div>";
  return h + "</div>";
}
function wireItemCardHtml(it, tab, idx, hit) {
  var h = "<div class='card' style='margin-top:10px'>";
  if (it.kind === "kev") h += "<span class='mono' style='font-size:11px;color:#f87171'>KEV ADDITION</span> ";
  else if (it.kind === "cve") h += "<span class='mono' style='font-size:11px;color:#f5b544'>NEW CVE</span> ";
  if (it.developing) h += "<span class='mono' style='font-size:11px;color:#34d399'>DEVELOPING</span> ";
  h += "<button class='ghost' data-wire-read='" + esc(encodeURIComponent(it.link)) + "' style='text-align:left;font-size:15px;min-height:40px'><b>" + (hit ? wireMarkTitle(it.title, hit) : esc(it.title)) + "</b></button>";
  h += "<div class='mono' style='font-size:11px;color:var(--dim)'>" + esc(it.sourceLabel || "") + (it.published ? " \u00b7 " + esc(ticLocal(it.published)) : "") + "</div>";
  if (hit) h += "<div class='mono' style='font-size:11px;color:var(--am)'>Matched: " + hit.map(function (t) { return t[0] === t[1] ? esc(t[0]) : esc(t[0]) + " \u2192 " + esc(t[1]); }).join(" \u00b7 ") + "</div>";
  if (it.excerpt) h += "<p style='font-size:13px;margin-top:6px'>" + esc(it.excerpt) + "</p>";
  if (tab) h += "<div class='row' style='margin-top:8px'><button class='ghost' data-wire-pull='" + tab + ":" + idx + "' aria-label='Pull this card into a case as reference' style='min-height:36px;padding:4px 12px;font-size:12px'>Pull into a case</button>"
    + "<button class='ghost' data-wire-start-case='" + esc(encodeURIComponent(it.link || "")) + "' aria-label='Start a case from this story' style='min-height:36px;padding:4px 12px;font-size:12px'>Start a case</button></div>";
  return h + "</div>";
}
function wireStartCase(link) {
  // v0.6.6e (#330): one act — the case is born from this
  // story: article shelved whole, a cited lead per CVE it
  // names, related coverage shelved, her briefing opening
  // the case chat with her recommended tasks staged as
  // proposals. The message names the bridge's accounting.
  if (!link) return;
  wirePullMsg = "Starting the case\u2026";
  render();
  fetch("/api/wire/start-case", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ link: link }) }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.ok && j.birth) {
      var b = j.birth;
      wirePullMsg = "Case started \u2014 \u201c" + (b.name || b.slug) + "\u201d: " + (b.leads || []).length + " cited lead" + ((b.leads || []).length === 1 ? "" : "s") + " filed, " + (b.related || []).length + " related stor" + ((b.related || []).length === 1 ? "y" : "ies") + " shelved, " + (b.staged || []).length + " task" + ((b.staged || []).length === 1 ? "" : "s") + " staged as proposals. Her briefing opens its case chat." + ((b.gaps && b.gaps.length) ? " Gaps: " + b.gaps.join(" ") : "");
      hydrateChatsFromServer();
    } else {
      wirePullMsg = "The case was not started \u2014 " + ((j && j.error) || "the bridge could not birth it.");
    }
    render();
  }).catch(function () { wirePullMsg = "The case was not started \u2014 the bridge did not answer."; render(); });
}
var wireFeedMsg = "";
function wireFeedsCardHtml(s) {
  // Your feeds (Build 3, #210): the operator's own RSS/Atom
  // sources. Validated by the bridge at add time — a bad feed
  // is refused BY NAME and never saved; a saved feed that
  // later fails names its failure on its pill in Feed health.
  // Feeds live in this install's config and do NOT ride Sync.
  var feeds = s.feeds || [];
  var h = "<div class='card' style='margin-top:12px'><b>Your feeds</b>";
  h += "<p style='font-size:13px;color:var(--dim);margin-top:4px'>Add any RSS or Atom feed and its stories flow into the News tab, badged with the name you give it, on the wire's normal cadence. A feed is checked before it is saved: unreachable, unparseable, or not-a-feed are refused by name. Feeds are per-install — they do not ride Sync.</p>";
  if (wireFeedMsg) h += "<p style='font-size:13px'>" + esc(wireFeedMsg) + "</p>";
  if (feeds.length) {
    feeds.forEach(function (f) {
      h += "<div style='display:flex;gap:8px;align-items:center;flex-wrap:wrap;margin-top:6px'><span><b>" + esc(f.name) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(f.url) + "</span></span><button class='ghost' data-wire-feed-remove='" + esc(f.id) + "' style='min-height:40px;padding:4px 10px'>Remove</button></div>";
    });
  } else {
    h += "<p style='color:var(--dim);font-size:13px'>None yet — the curated sources above are the wire's own.</p>";
  }
  h += "<div class='row' style='margin-top:8px;gap:8px;flex-wrap:wrap'><input id='wire-feed-name' type='text' placeholder='Feed name — e.g. My Lab Blog' aria-label='Name for the new feed' style='min-height:44px;flex:1 1 160px'><input id='wire-feed-url' type='url' placeholder='https://example.org/feed.xml' aria-label='URL of the new feed' style='min-height:44px;flex:2 1 240px'><button class='act' id='wire-feed-add' style='min-height:44px'>Add feed</button></div>";
  return h + "</div>";
}
function wireTabsHtml() {
  // The Wire's tab bar (v0.4.1e, #230): the four families as
  // PAGES inside The Wire — News, CVEs, KEVs, CTFs — one tap
  // apart at the top of every family page, each tab carrying
  // its family's count from the stores the page already
  // holds. The selected tab is intelTab, persisted across
  // reloads by setIntelTab, so the tab you left is the tab
  // you return to. The Center and Bounties are not wire
  // families — they keep their own doors (Home's cards, the
  // nav) and no tab here pretends otherwise.
  var counts = { news: 0, cve: 0, kev: 0, ctfs: 0, pulses: 0 };
  if (pulseState && pulseState.watches) counts.pulses = pulseState.watches.length;
  if (wireState && wireState.items) wireState.items.forEach(function (it) { if (counts[it.kind] !== undefined) counts[it.kind]++; });
  if (ctfState) counts.ctfs = (ctfState.running || []).length + (ctfState.upcoming || []).length;
  var tabs = [["news", "News"], ["cve", "CVEs"], ["kev", "KEVs"], ["ctfs", "CTFs"], ["pulses", "Pulses"]];
  var h = "<div class='wiretabs' role='tablist' aria-label='The Wire — families'>";
  tabs.forEach(function (t) {
    var cur = intelTab === t[0];
    h += "<button type='button' role='tab' aria-selected='" + (cur ? "true" : "false") + "' class='" + (cur ? "act" : "ghost") + "' data-tic-go='" + t[0] + "' aria-label='The Wire — " + t[1] + "'>" + t[1] + " <span class='wt-count'>" + counts[t[0]] + "</span></button>";
  });
  return h + "</div>";
}
// ---------- Wire tab search + pull into a case (v0.5.10e, #283) ----------
// Every Wire tab carries its own search bar, searching its
// own corpus: fuzzy — token-based, typo-tolerant, relevance-
// ranked — with the match visible on the card (the matched
// words lit in the title, the matched terms named under it).
// Clearing the search restores the tab's standing order.
// Every card also carries Pull into a case: the card lands on
// a case chat's REFERENCE SHELF (bridge-side, /api/cases/
// shelf) — reference, not a finding; the case file is never
// touched by a pull.
var wireTabSearch = { news: "", cve: "", kev: "", ctfs: "" };
var wireTabLists = { news: [], cve: [], kev: [], ctfs: [] };
var wirePullPick = null, wirePullMsg = "";
var WIRE_TAB_LABELS = { news: "News", cve: "CVEs", kev: "KEVs", ctfs: "CTFs" };
function wireLev(a, b) {
  var m = a.length, n = b.length;
  if (Math.abs(m - n) > 2) return 99;
  var prev = [], cur = [], i, j;
  for (j = 0; j <= n; j++) prev[j] = j;
  for (i = 1; i <= m; i++) {
    cur[0] = i;
    for (j = 1; j <= n; j++) {
      cur[j] = Math.min(prev[j] + 1, cur[j - 1] + 1, prev[j - 1] + (a[i - 1] === b[j - 1] ? 0 : 1));
    }
    var t = prev; prev = cur; cur = t;
  }
  return prev[n];
}
function wireFuzzyRank(query, rows) {
  // rows: [{title, body}] in standing order. Returns null when
  // no search is active; else the matching rows ranked —
  // [{i, score, terms:[[queryToken, matchedWord], …]}] — a row
  // matches only when EVERY query token finds a home in it.
  var qtoks = String(query || "").toLowerCase().split(/[^a-z0-9._-]+/).filter(Boolean);
  if (!qtoks.length) return null;
  var out = [];
  rows.forEach(function (row, idx) {
    var titleWords = String(row.title || "").toLowerCase().split(/[^a-z0-9._-]+/).filter(Boolean);
    var bodyWords = String(row.body || "").toLowerCase().split(/[^a-z0-9._-]+/).filter(Boolean);
    var score = 0, terms = [], all = true;
    qtoks.forEach(function (qt) {
      var best = 0, bw = "";
      function consider(w, base) {
        var s = 0;
        if (w === qt) s = base + 3;
        else if (qt.length >= 3 && w.indexOf(qt) === 0) s = base + 2;
        else if (w.indexOf(qt) !== -1) s = base + 1;
        else if (qt.length >= 4 && wireLev(qt, w) <= (qt.length >= 8 ? 2 : 1)) s = base + 2;
        if (s > best) { best = s; bw = w; }
      }
      titleWords.forEach(function (w) { consider(w, 3); });
      if (!best) bodyWords.forEach(function (w) { consider(w, 0); });
      if (!best) all = false; else { score += best; terms.push([qt, bw]); }
    });
    if (all) out.push({ i: idx, score: score, terms: terms });
  });
  out.sort(function (a, b) { return b.score - a.score || a.i - b.i; });
  return out;
}
function wireMarkTitle(title, terms) {
  if (!terms || !terms.length) return esc(title);
  var hitWords = {};
  terms.forEach(function (t) { hitWords[t[1]] = 1; });
  return String(title).split(/(\s+)/).map(function (piece) {
    var w = piece.toLowerCase().replace(/[^a-z0-9._-]/g, "");
    if (w && hitWords[w]) return "<mark class='wire-hit'>" + esc(piece) + "</mark>";
    return esc(piece);
  }).join("");
}
function wireSearchBarHtml(tab) {
  var q = wireTabSearch[tab] || "";
  var h = "<div class='card wiresearch' style='margin-top:12px'><div class='row' style='gap:8px;flex-wrap:wrap;align-items:center'>"
    + "<input id='wire-search-" + tab + "' data-wire-search='" + tab + "' type='search' value='" + esc(q) + "' placeholder='Search " + WIRE_TAB_LABELS[tab] + " \u2014 fuzzy; typos forgiven' aria-label='Search the " + WIRE_TAB_LABELS[tab] + " tab' style='flex:1 1 220px'>"
    + "<button class='ghost' id='wire-search-clear-" + tab + "' data-wire-search-clear='" + tab + "' aria-label='Clear the " + WIRE_TAB_LABELS[tab] + " search' style='min-height:44px" + (q ? "" : ";display:none") + "'>Clear</button></div>"
    + "<p style='font-size:12px;color:var(--dim);margin-top:6px'>Searches this tab only. Clearing brings the standing order back.</p></div>";
  return h;
}
function wireResultsHtml(tab) {
  return tab === "ctfs" ? wireCtfCardsHtml() : wireWireCardsHtml(tab);
}
function wireWireCardsHtml(tab) {
  var items = (wireState.items || []).filter(function (it) { return it.kind === tab; }).slice(0, 60);
  wireTabLists[tab] = items.map(function (it) {
    return { tab: tab, title: it.title || "", link: it.link || "", source: it.sourceLabel || "",
             text: (it.title || "") + "\n" + (it.sourceLabel || "") + (it.published ? " \u00b7 " + ticLocal(it.published) : "") + "\n\n" + (it.excerpt || "") };
  });
  var ranked = wireFuzzyRank(wireTabSearch[tab], wireTabLists[tab].map(function (c) { return { title: c.title, body: c.text }; }));
  var h = "";
  if (ranked) {
    h += "<p class='mono' style='font-size:12px;color:var(--dim);margin-top:10px'>" + ranked.length + " of " + items.length + " cards match \u201c" + esc(wireTabSearch[tab]) + "\u201d \u2014 closest first.</p>";
    if (!ranked.length) h += "<div class='card' style='margin-top:10px'><p style='color:var(--dim)'>No card on this tab answers to that \u2014 the absence is this tab's own: nothing here matches, however loosely. Try fewer or different words.</p></div>";
    ranked.forEach(function (r) { h += wireItemCardHtml(items[r.i], tab, r.i, r.terms); });
    return h;
  }
  if (!items.length) {
    return h + "<div class='card' style='margin-top:12px'><p style='color:var(--dim)'>" + (tab === "news"
      ? "No news on the wire yet \u2014 the first fetch brings the headlines. A feed that does not answer is named in Feed health above, never silently missing."
      : "Nothing in this family on the wire right now.") + "</p></div>";
  }
  items.forEach(function (it, i) { h += wireItemCardHtml(it, tab, i, null); });
  return h;
}
function ctfCardText(e) {
  var paras = [(e.format || "Format unlisted") + " \u00b7 " + (e.onsite ? "On-site \u2014 " + (e.location || "place unlisted") : "Online") + (e.weight ? " \u00b7 weight " + e.weight : "") + (e.restrictions ? " \u00b7 " + e.restrictions : ""),
               ticLocal(e.start) + " \u2192 " + ticLocal(e.finish) + " (your timezone)"];
  if (e.organizers && e.organizers.length) paras.push("Run by " + e.organizers.join(", ") + ".");
  if (e.description) paras.push(e.description);
  if (e.prizes) paras.push("Prizes: " + e.prizes);
  return (e.title || "") + "\n" + paras.join("\n");
}
function wireCtfCardsHtml() {
  var s = ctfState;
  var running = s.running || [], upcoming = (s.upcoming || []).slice(0, 30);
  var all = running.concat(upcoming);
  wireTabLists.ctfs = all.map(function (e) {
    return { tab: "ctfs", title: e.title || "", link: e.ctftimeUrl || e.url || "", source: "CTFtime", text: ctfCardText(e) };
  });
  var ranked = wireFuzzyRank(wireTabSearch.ctfs, wireTabLists.ctfs.map(function (c) { return { title: c.title, body: c.text }; }));
  var h = "";
  if (ranked) {
    h += "<p class='mono' style='font-size:12px;color:var(--dim);margin-top:10px'>" + ranked.length + " of " + all.length + " events match \u201c" + esc(wireTabSearch.ctfs) + "\u201d \u2014 closest first.</p>";
    if (!ranked.length) h += "<div class='card' style='margin-top:10px'><p style='color:var(--dim)'>No event on the board answers to that. Try fewer or different words.</p></div>";
    ranked.forEach(function (r) { h += ctfEventHtml(all[r.i], r.i, r.terms); });
    return h;
  }
  h += "<h3>Running now</h3>";
  if (!running.length) h += "<div class='card'><p style='color:var(--dim)'>Nothing is running right now.</p></div>";
  running.forEach(function (e, i) { h += ctfEventHtml(e, i, null); });
  h += "<h3>Upcoming</h3>";
  if (!upcoming.length) h += "<div class='card'><p style='color:var(--dim)'>Nothing upcoming in the window the board carries.</p></div>";
  upcoming.forEach(function (e, i) { h += ctfEventHtml(e, running.length + i, null); });
  return h;
}
function wirePullPickerHtml() {
  var c = wirePullPick;
  var h = "<div id='wire-pull-shade'></div>";
  h += "<div id='wire-pull-picker' role='dialog' aria-modal='true' aria-label='Pull this card into a case'>";
  h += "<div class='mono' style='font-size:11px;color:var(--dim)'>PULL INTO A CASE \u2014 AS REFERENCE</div>";
  h += "<h3 style='margin:6px 0'>" + esc(c.title) + "</h3>";
  h += "<div class='mono' style='font-size:11px;color:var(--dim)'>" + esc(WIRE_TAB_LABELS[c.tab] || c.tab) + " \u00b7 " + esc(c.source || "") + "</div>";
  h += "<p style='font-size:13px;margin-top:8px'>The card lands on the case chat's <b>reference shelf</b> \u2014 its full text, link and provenance \u2014 for Navi to pull from as she works the case. It is <b>not a finding</b>: the case file is untouched, and filing stays a separate act.</p>";
  if (!casesState) h += "<p style='color:var(--dim);font-size:13px'>Gathering your cases\u2026</p>";
  else if (!casesState.length) h += "<p style='color:var(--dim);font-size:13px'>No cases yet \u2014 a card needs a case to land in. Start one in Case Files first.</p>";
  else {
    h += "<div style='display:flex;flex-direction:column;gap:6px;margin-top:10px'>";
    casesState.forEach(function (cs) {
      h += "<button class='ghost' data-wire-pull-case='" + esc(cs.slug) + "' style='min-height:44px;text-align:left'><b>" + esc(cs.name || cs.slug) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(cs.slug) + "</span></button>";
    });
    h += "</div>";
  }
  h += "<div class='row' style='margin-top:12px'><button class='ghost' id='wire-pull-cancel' style='min-height:40px'>Cancel</button></div>";
  return h + "</div>";
}
function wireDoPull(slug) {
  var card = wirePullPick;
  if (!card || !slug) return;
  var name = slug;
  (casesState || []).forEach(function (c) { if (c.slug === slug) name = c.name || slug; });
  casePost("/api/cases/shelf", { case: slug, act: "add", card: { tab: card.tab, title: card.title, link: card.link, source: card.source, text: card.text } }, function (j) {
    wirePullPick = null;
    if (j && j.ok) {
      wirePullMsg = j.note === "duplicate"
        ? "\u201c" + card.title + "\u201d is already on \u201c" + name + "\u201d's reference shelf \u2014 as " + ((j.ref && j.ref.id) || "") + ". No second copy was made."
        : "Pulled onto \u201c" + name + "\u201d's reference shelf as " + ((j.ref && j.ref.id) || "") + " \u2014 reference, not a finding. It stands in the case's chat, and Navi can read it there.";
      hydrateChatsFromServer();
    } else {
      wirePullMsg = "The pull did not land \u2014 " + ((j && j.error) || "the bridge could not shelf it.");
    }
    render();
  });
}
var shelfClosed = {};
try { shelfClosed = JSON.parse(localStorage.getItem("navi.shelfClosed") || "{}") || {}; } catch (e) { shelfClosed = {}; }
function shelfIsClosed(slug) { return !!(slug && shelfClosed[slug]); }
function shelfToggle(slug) {
  if (!slug) return;
  if (shelfClosed[slug]) delete shelfClosed[slug]; else shelfClosed[slug] = true;
  try { localStorage.setItem("navi.shelfClosed", JSON.stringify(shelfClosed)); } catch (e) {}
  render();
}
function caseShelfHtml(ac) {
  // The reference shelf, standing at the head of the case's
  // room — labeled as what it is, distinct from messages and
  // from findings: reference, not findings. Removal is
  // shelf-local and says so.
  // v0.6.7e (#333): the shelf COLLAPSES. Its header row is a
  // real button — tap it and the shelf folds to the header
  // alone; tap again and it stands back up. The state is
  // remembered PER CASE in this seat's view state
  // (localStorage navi.shelfClosed — view state only, never
  // written into the case file), so it survives reload and
  // one case's fold never folds another's. Collapsing is a
  // view act only: the items, their Remove buttons and
  // Navi's reading of the shelf are exactly as they were,
  // and nothing is removed or filed by folding.
  if (!ac || !ac.case || !ac.shelf || !ac.shelf.length) return "";
  var closed = shelfIsClosed(ac.case);
  var h = "<div class='card shelf-card' style='margin-top:10px'>"
    + "<button type='button' class='ghost shelf-toggle' data-shelf-toggle='" + esc(ac.case) + "' aria-expanded='" + (closed ? "false" : "true") + "' aria-label='Toggle this case's reference shelf' style='min-height:44px;width:100%;text-align:left'>"
    + "<span class='mono' style='font-size:12px'><b>REFERENCE SHELF</b> <span style='color:var(--dim)'>pulled from the Wire \u00b7 reference, not case findings \u00b7 " + ac.shelf.length + (ac.shelf.length === 1 ? " item" : " items") + "</span></span>"
    + "<span style='float:right' aria-hidden='true'>" + (closed ? "\u25b8" : "\u25be") + "</span></button>";
  if (closed) return h + "</div>";
  ac.shelf.forEach(function (r) {
    h += "<div style='display:flex;gap:8px;align-items:flex-start;margin-top:8px'><div style='flex:1;min-width:0'><span class='mono' style='font-size:11px;color:var(--am)'>" + esc(r.id) + "</span> "
      + (r.link ? "<a href='" + esc(r.link) + "' target='_blank' rel='noopener' style='color:var(--cy)'>" + esc(r.title) + "</a>" : "<b>" + esc(r.title) + "</b>")
      + "<div class='mono' style='font-size:11px;color:var(--dim)'>" + esc(WIRE_TAB_LABELS[r.tab] || r.tab || "") + (r.source ? " \u00b7 " + esc(r.source) : "") + " \u00b7 pulled by " + (r.pulledBy === "navi" ? "Navi" : "you") + (r.pulledAt ? " \u00b7 " + esc(new Date(r.pulledAt * 1000).toLocaleString()) : "") + "</div></div>"
      + "<button class='ghost' data-shelf-remove='" + esc(r.id) + "' data-shelf-case='" + esc(ac.case) + "' aria-label='Remove reference " + esc(r.id) + " from this case's shelf' style='min-height:36px;padding:4px 10px;font-size:12px'>Remove</button></div>";
  });
  h += "<div style='font-size:12px;color:var(--dim);margin-top:8px'>Navi reads these in this room \u2014 name one (\u201cuse reference R1\u201d) or ask her to pull it into a reply. Removing one touches only this shelf.</div>";
  return h + "</div>";
}
function refPullsHtml(m) {
  // The receipt for her shelf pulls — built from the bridge's
  // record of what was actually placed before her.
  if (!m.refPulls || !m.refPulls.length) return "";
  return m.refPulls.map(function (p) {
    return "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Read from the shelf</b> <span style='color:var(--dim)'>her pull, receipted by the bridge</span></div>"
      + "<div style='font-size:13px;margin-top:2px'><span class='mono' style='font-size:11px;color:var(--am)'>" + esc(p.id) + "</span> \u201c" + esc(p.title) + "\u201d \u2014 cited as reference above</div></div>";
  }).join("");
}
function wireTabHtml() {
  var h = "";
  if (wireMsg) h += "<p style='font-size:13px'>" + esc(wireMsg) + "</p>";
  if (!wireState) { if (!wireFetching) refreshWire(); return h + "<div class='card'>Gathering the wire\u2026</div>"; }
  var s = wireState;
  if (s.ok === false) return h + "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(s.error || "The wire is unavailable.") + "</div>";
  // Feed health lives here now (Build 4, #213): the pill field
  // came off Home and sits at the top of the News view \u2014 its
  // collapsed slim-row form, per-operator persisted state, and
  // house motion carried over unchanged.
  h += wirePillsHtml();
  h += "<div class='card' style='margin-top:12px'><div style='display:flex;flex-wrap:wrap;gap:8px;align-items:center;justify-content:space-between'><b>News — the wire</b><button class='act' id='wire-pull' style='min-height:40px'" + (s.pulling ? " disabled" : "") + ">" + (s.pulling ? "Fetching\u2026" : "Fetch now") + "</button></div>";
  h += "<p style='font-size:13px;margin-top:6px'>" + esc(s.staleness || "") + "</p>";
  h += "<p style='font-size:12px;color:var(--dim)'>News only on this lane — the wire's CVE items live on the CVE tab, its KEV additions on the KEV tab, and feed health lives in the field at the top of this page.</p>";
  h += "<div class='row' style='margin-top:8px;gap:14px;align-items:center;flex-wrap:wrap'><label style='display:flex;gap:8px;align-items:center;min-height:40px'><input type='checkbox' id='wire-enabled'" + (s.enabled ? " checked" : "") + "> Wire enabled</label>";
  h += "<label style='font-size:12px;color:var(--dim)'>Fetch every <select id='wire-interval' style='min-height:40px'>" + [15, 30, 60, 120].map(function (n) { return "<option value='" + n + "'" + (Number(s.intervalMinutes) === n ? " selected" : "") + ">" + n + " min</option>"; }).join("") + "</select></label>";
  h += "<label style='font-size:12px;color:var(--dim)'>Keep <select id='wire-retention' style='min-height:40px'>" + [3, 7, 14, 30].map(function (n) { return "<option value='" + n + "'" + (Number(s.retentionDays) === n ? " selected" : "") + ">" + n + " days</option>"; }).join("") + "</select></label>";
  h += "<button class='ghost' id='wire-save' style='min-height:40px'>Save wire settings</button></div></div>";
  h += wireFeedsCardHtml(s);
  // News is news only (Build 3, #210): the lane filter is the
  // item's own kind — CVE and KEV items are re-laned to their
  // tabs, never dropped. v0.5.10e (#283): the tab's search bar
  // stands over the feed, and the results container carries
  // the cards so typing re-renders only the list.
  h += wireSearchBarHtml("news");
  h += "<div id='wire-results-news'>" + wireWireCardsHtml("news") + "</div>";
  // The wire's other families are one tap away on the tab
  // bar at the top of this view (v0.4.1e, #230) — the Build 4
  // door row that stood here is retired; the bar supersedes it.
  return h;
}
function wireKindHtml(kind) {
  // The CVE and KEV tabs (Build 3, #210): the wire's own items
  // of one family, with the same cards and the same reader the
  // News lane always had. The Center's graded queue is a
  // different store — it lives on the Center tab, and the KEV
  // header carries its count so the two never blur.
  var h = "";
  if (!wireState) { if (!wireFetching) refreshWire(); return h + "<div class='card'>Gathering the wire\u2026</div>"; }
  var s = wireState;
  if (s.ok === false) return h + "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(s.error || "The wire is unavailable.") + "</div>";
  var items = (s.items || []).filter(function (it) { return it.kind === kind; });
  if (kind === "cve") {
    h += "<div class='card'><b>New CVEs — from the wire (NVD)</b><p style='font-size:13px;color:var(--dim);margin-top:4px'>" + items.length + " on the wire right now. These are the NVD's newest, as headlines, and each opens in the reader. This page stands on its own inside The Wire — the Center's graded queue (CVSS, EPSS, matched records) is its own surface, reached from Home.</p></div>";
  } else {
    h += "<div class='card'><b>KEV additions — known-exploited, from the wire</b><p style='font-size:13px;color:var(--dim);margin-top:4px'>" + items.length + " additions on the wire right now" + (intelState ? " \u00b7 the Center's catalog holds " + esc(intelState.kevTotal || 0) + " known-exploited in all" : "") + ". Each card opens the CISA catalog entry through the reader; where extraction cannot work, the fallback is named, never faked.</p></div>";
  }
  // v0.5.10e (#283): the tab's own search bar + results.
  h += wireSearchBarHtml(kind);
  h += "<div id='wire-results-" + kind + "'>" + wireWireCardsHtml(kind) + "</div>";
  return h;
}
function openWireReader(link) {
  if (wireReaderBusy) return;
  wireReaderBusy = true;
  fetch("/api/wire/article?url=" + encodeURIComponent(link)).then(function (r) { return r.json(); }).then(function (j) {
    wireReaderBusy = false;
    if (j && j.ok) { wireReader = j; render(); return; }
    // The named fallback (Addition 4): a failed extraction is
    // never dressed as the article — desktop opens the
    // original in a new tab; mobile opens the article proper.
    var sentence = (j && j.sentence) || "The reader could not extract that article.";
    if (window.innerWidth <= 760) { wireMsg = sentence; window.location.href = link; return; }
    wireMsg = sentence + " Opened the original in a new tab instead.";
    window.open(link, "_blank");
    render();
  }).catch(function () {
    wireReaderBusy = false;
    wireMsg = "The reader could not reach the bridge for that article.";
    render();
  });
}
function bindWireReaderClose() {
  // The reader is portaled to <body> while it is open (Build
  // 10, #196), so its close controls are bound here, by id —
  // one binder, called from the wiring pass and from the
  // portal step in render(), so the two never disagree.
  var c = document.getElementById("wire-reader-close");
  if (c) c.onclick = function () { wireReader = null; render(); };
  var t = document.getElementById("wire-reader-close-top");
  if (t) t.onclick = function () { wireReader = null; render(); };
  var s = document.getElementById("wire-reader-shade");
  if (s) s.onclick = function () { wireReader = null; render(); };
}
function wireReaderHtml() {
  var j = wireReader, it = j.item || {};
  var h = "<div id='wire-reader-shade'></div>";
  h += "<div id='wire-reader' role='dialog' aria-modal='true' aria-label='Article reader'>";
  // Build 11 (#196 rider): the bar is its own flex child of the
  // panel and the article scrolls in the .wire-reader-body box
  // below it — no sticky header inside the scroll box, so the
  // bar can never ride over or clip the headline (see the
  // #wire-reader CSS note).
  h += "<div class='wire-reader-head'><div style='display:flex;gap:8px;align-items:flex-start;justify-content:space-between'><span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(it.source || "") + (it.published ? " · " + esc(ticLocal(it.published)) : "") + (j.kind === "record" ? " · read in Apex — the listing as the board carries it" : " · read in Apex, extracted from the original") + "</span><button class='ghost' id='wire-reader-close-top' aria-label='Close the article reader' style='min-height:40px;flex:0 0 auto'>Close</button></div></div>";
  h += "<div class='wire-reader-body'>";
  h += "<h3 style='margin:8px 0'>" + esc(j.title || it.title || "Article") + "</h3>";
  (j.paragraphs || []).forEach(function (p) { h += "<p style='font-size:14px;line-height:1.55;margin-top:8px'>" + esc(p) + "</p>"; });
  h += "<div class='row' style='margin-top:14px;gap:8px'><a class='act' href='" + esc(it.link || "#") + "' target='_blank' rel='noopener' style='min-height:40px;text-decoration:none;display:inline-flex;align-items:center;padding:0 14px'>Open original</a><button class='ghost' id='wire-reader-close' style='min-height:40px'>Close</button></div>";
  return h + "</div></div>";
}
function ctfEventByIdx(i) {
  // The board's combined order (running, then upcoming) —
  // the same order ctfTabHtml and Home's "Next CTFs" card
  // render in, so an index names one event everywhere.
  if (!ctfState) return null;
  return (ctfState.running || []).concat(ctfState.upcoming || [])[i] || null;
}
function openCtfReader(e) {
  // CTF information opens in the same reader as every other
  // family (v0.4.1e, #230 — one reading surface, uniform):
  // the event's listing as the board carries it, composed
  // from the fields the board already holds. The reader's
  // Open-original fallback reaches the event's own page,
  // exactly as it does for the wire's items.
  if (!e) return;
  var paras = [];
  paras.push((e.format || "Format unlisted") + " · " + (e.onsite ? "On-site — " + (e.location || "place unlisted") : "Online") + (e.weight ? " · weight " + e.weight : "") + (e.restrictions ? " · " + e.restrictions : ""));
  paras.push(ticLocal(e.start) + " → " + ticLocal(e.finish) + " (your timezone)");
  if (e.organizers && e.organizers.length) paras.push("Run by " + e.organizers.join(", ") + ".");
  if (e.description) paras.push(e.description);
  if (e.prizes) paras.push("Prizes: " + e.prizes);
  wireReader = { kind: "record", item: { source: "CTFtime — the board's listing", link: e.ctftimeUrl || e.url || "", published: e.start }, title: e.title, paragraphs: paras };
  render();
}
function ctfEventHtml(e, idx, hit) {
  var h = "<div class='card' style='margin-top:10px'><div style='display:flex;gap:12px;align-items:flex-start'>";
  if (e.logo) h += "<img src='" + esc(e.logo) + "' alt='' style='width:44px;height:44px;object-fit:contain;border-radius:8px;background:var(--panel2)'>";
  h += "<div style='flex:1'><button type='button' class='ghost tic-ev-read' data-ctf-read='" + idx + "' aria-label='Read this event in the reader' style='text-align:left;font-size:15px;min-height:40px'><b>" + (hit ? wireMarkTitle(e.title, hit) : esc(e.title)) + "</b></button>";
  if (hit) h += "<div class='mono' style='font-size:11px;color:var(--am)'>Matched: " + hit.map(function (t) { return t[0] === t[1] ? esc(t[0]) : esc(t[0]) + " \u2192 " + esc(t[1]); }).join(" \u00b7 ") + "</div>";
  h += "<div class='mono' style='font-size:11px;color:var(--dim)'>" + esc(e.format || "format unlisted") + " · " + (e.onsite ? "on-site · " + esc(e.location || "") : "online") + (e.weight ? " · weight " + esc(e.weight) : "") + (e.restrictions ? " · " + esc(e.restrictions) : "") + "</div>";
  h += "<div class='mono' style='font-size:12px;margin-top:2px'>" + esc(ticLocal(e.start)) + " → " + esc(ticLocal(e.finish)) + " <span style='color:var(--dim)'>(your timezone)</span></div>";
  if (e.organizers && e.organizers.length) h += "<div style='font-size:12px;color:var(--dim);margin-top:2px'>by " + esc(e.organizers.join(", ")) + "</div>";
  if (e.description) h += "<p style='font-size:13px;margin-top:6px'>" + esc(e.description) + "</p>";
  if (e.prizes) h += "<p style='font-size:13px;margin-top:6px'><b>Prizes:</b> " + esc(e.prizes) + "</p>";
  h += "<div class='row' style='margin-top:8px;gap:8px'>";
  if (e.ctftimeUrl) h += "<a class='ghost' href='" + esc(e.ctftimeUrl) + "' target='_blank' rel='noopener' style='min-height:36px;text-decoration:none;display:inline-flex;align-items:center;padding:0 12px'>CTFtime listing</a>";
  if (e.url) h += "<a class='ghost' href='" + esc(e.url) + "' target='_blank' rel='noopener' style='min-height:36px;text-decoration:none;display:inline-flex;align-items:center;padding:0 12px'>Event site</a>";
  h += "<button type='button' class='ghost' data-wire-pull='ctfs:" + idx + "' aria-label='Pull this event into a case as reference' style='min-height:36px;padding:4px 12px;font-size:12px'>Pull into a case</button>";
  return h + "</div></div></div></div>";
}
function ctfTabHtml() {
  var h = "";
  if (ctfMsg) h += "<p style='font-size:13px'>" + esc(ctfMsg) + "</p>";
  if (!ctfState) { if (!ctfFetching) refreshCtf(); return h + "<div class='card'>Gathering the board…</div>"; }
  var s = ctfState;
  if (s.ok === false) return h + "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(s.error || "The board is unavailable.") + "</div>";
  h += "<div class='card'><div style='display:flex;flex-wrap:wrap;gap:8px;align-items:center;justify-content:space-between'><b>The CTF board</b><button class='act' id='ctf-pull' style='min-height:40px'" + (s.pulling ? " disabled" : "") + ">" + (s.pulling ? "Fetching…" : "Fetch now") + "</button></div>";
  h += "<p style='font-size:13px;margin-top:6px'>" + esc(s.staleness || "") + "</p>";
  h += "<label style='display:flex;gap:8px;align-items:center;min-height:40px'><input type='checkbox' id='ctf-enabled'" + (s.enabled ? " checked" : "") + "> Board enabled</label></div>";
  // v0.5.10e (#283): the board's own search bar + results.
  h += wireSearchBarHtml("ctfs");
  h += "<div id='wire-results-ctfs'>" + wireCtfCardsHtml() + "</div>";
  return h;
}
function bountiesTabHtml() {
  var h = "";
  if (bountyMsg) h += "<p style='font-size:13px'>" + esc(bountyMsg) + "</p>";
  if (!bountyState) { if (!bountyFetching) refreshBounties(); return h + "<div class='card'>Gathering the programs…</div>"; }
  var s = bountyState;
  if (s.ok === false) return h + "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(s.error || "The bounty board is unavailable.") + "</div>";
  h += "<div class='card'><div style='display:flex;flex-wrap:wrap;gap:8px;align-items:center;justify-content:space-between'><b>Bug bounties — public programs</b><button class='act' id='bounty-pull' style='min-height:40px'" + (s.pulling ? " disabled" : "") + ">" + (s.pulling ? "Fetching…" : "Fetch now") + "</button></div>";
  h += "<p style='font-size:13px;margin-top:6px'>" + esc(s.staleness || "") + "</p>";
  h += "<p style='font-size:13px'>" + esc((s.programs || []).length) + " programs in the Chaos index · " + esc(s.payingCount || 0) + " bounty-paying.</p>";
  h += "<label style='display:flex;gap:8px;align-items:center;min-height:40px'><input type='checkbox' id='bounty-enabled'" + (s.enabled ? " checked" : "") + "> Board enabled</label></div>";
  (s.programs || []).slice(0, 120).forEach(function (p) {
    h += "<div class='card' style='margin-top:10px'><b style='font-size:15px'>" + esc(p.name) + "</b> ";
    if (p.isNew) h += "<span class='mono' style='font-size:11px;color:#34d399'>NEW</span> ";
    h += "<span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(p.platform || "platform unlisted") + "</span>";
    h += "<div style='font-size:13px;margin-top:4px'>" + (p.bounty ? "<span style='color:#34d399'>Pays bounties</span>" : "<span style='color:var(--dim)'>No bounty listed</span>")
      + (p.scopeSize ? " · scope: " + esc(p.scopeSize) + " domains in the Chaos dataset" : "")
      + " · Safe harbor: not stated by the index</div>";
    if (p.lastUpdated) h += "<div class='mono' style='font-size:11px;color:var(--dim)'>index updated " + esc(String(p.lastUpdated).slice(0, 10)) + "</div>";
    h += "<div class='row' style='margin-top:6px'><a class='ghost' href='" + esc(p.link) + "' target='_blank' rel='noopener' style='min-height:36px;text-decoration:none;display:inline-flex;align-items:center;padding:0 12px'>Program page</a></div></div>";
  });
  if ((s.programs || []).length > 120) h += "<p style='color:var(--dim);font-size:12px;margin-top:8px'>Showing the first 120 of " + esc(s.programs.length) + " — bounty-paying programs sort first.</p>";
  h += "<div class='card' style='margin-top:12px'><h3 style='margin-top:0'>Disclosed reports</h3>";
  if (s.reports && s.reports.length) {
    s.reports.forEach(function (r) { h += "<div style='margin-top:6px'><a href='" + esc(r.link || "#") + "' target='_blank' rel='noopener' style='color:var(--cy)'>" + esc(r.title || "") + "</a> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(r.program || "") + (r.severity ? " · " + esc(r.severity) : "") + "</span></div>"; });
  } else {
    h += "<p style='color:var(--dim);font-size:13px'>" + esc(s.reportsSentence || "No disclosed-reports feed is wired.") + "</p>";
  }
  return h + "</div>";
}
function wireAwareness() {
  if (view === "CTFBoard" && intelTab !== "ctfs") setIntelTab("ctfs");
  mainEl.querySelectorAll("[data-tic-go]").forEach(function (b) { b.onclick = function () { setIntelTab(b.getAttribute("data-tic-go")); render(); }; });
  mainEl.querySelectorAll("[data-ctf-read]").forEach(function (b) { b.onclick = function () { openCtfReader(ctfEventByIdx(Number(b.getAttribute("data-ctf-read")))); }; });
  mainEl.querySelectorAll("[data-wire-read]").forEach(function (b) { b.onclick = function () { openWireReader(decodeURIComponent(b.getAttribute("data-wire-read"))); }; });
  // Tab search (v0.5.10e, #283): typing updates only the
  // tab's results container — the input itself is never
  // re-rendered, so focus and the caret never move.
  mainEl.querySelectorAll("[data-wire-search]").forEach(function (inp) { inp.oninput = function () {
    var tab = inp.getAttribute("data-wire-search");
    wireTabSearch[tab] = inp.value;
    var box = document.getElementById("wire-results-" + tab);
    if (box) { box.innerHTML = wireResultsHtml(tab); wireAwareness(); }
    var clr = document.getElementById("wire-search-clear-" + tab);
    if (clr) clr.style.display = inp.value ? "" : "none";
  }; });
  mainEl.querySelectorAll("[data-wire-search-clear]").forEach(function (b) { b.onclick = function () {
    var tab = b.getAttribute("data-wire-search-clear");
    wireTabSearch[tab] = "";
    var inp = document.getElementById("wire-search-" + tab);
    if (inp) inp.value = "";
    b.style.display = "none";
    var box = document.getElementById("wire-results-" + tab);
    if (box) { box.innerHTML = wireResultsHtml(tab); wireAwareness(); }
    if (inp) inp.focus();
  }; });
  // Pull into a case (v0.5.10e, #283): the pick opens the
  // case picker; the choice posts to the shelf's door.
  mainEl.querySelectorAll("[data-wire-pull]").forEach(function (b) { b.onclick = function () {
    var parts = (b.getAttribute("data-wire-pull") || "").split(":");
    var card = (wireTabLists[parts[0]] || [])[Number(parts[1])];
    if (!card) return;
    wirePullPick = card; wirePullMsg = "";
    if (!casesState) {
      fetch("/api/cases").then(function (r) { return r.json(); }).then(function (j) { casesState = (j && j.cases) || []; if (wirePullPick) render(); }).catch(function () { casesState = []; if (wirePullPick) render(); });
    }
    render();
  }; });
  mainEl.querySelectorAll("[data-wire-pull-case]").forEach(function (b) { b.onclick = function () { wireDoPull(b.getAttribute("data-wire-pull-case")); }; });
  mainEl.querySelectorAll("[data-wire-start-case]").forEach(function (b) { b.onclick = function () { try { wireStartCase(decodeURIComponent(b.getAttribute("data-wire-start-case") || "")); } catch (e) { wireStartCase(b.getAttribute("data-wire-start-case") || ""); } }; });
  var bind = function (id, fn) { var el = document.getElementById(id); if (el) el.onclick = fn; };
  bind("wire-pull-cancel", function () { wirePullPick = null; render(); });
  var pullShade = document.getElementById("wire-pull-shade");
  if (pullShade) pullShade.onclick = function () { wirePullPick = null; render(); };
  bindWireReaderClose();
  bind("wire-pills-toggle", function () {
    wirePillsOpen = !wirePillsOpen;
    try { localStorage.setItem("navi.wirePills", wirePillsOpen ? "open" : "closed"); } catch (e) {}
    render();
  });
  bind("wire-feed-add", function () {
    var nm = document.getElementById("wire-feed-name");
    var ur = document.getElementById("wire-feed-url");
    var name = nm ? nm.value.trim() : "";
    var url = ur ? ur.value.trim() : "";
    if (!name) { wireFeedMsg = "Give the feed a name first."; render(); return; }
    if (!url) { wireFeedMsg = "Give the feed a URL first."; render(); return; }
    wireFeedMsg = "Checking the feed\u2026";
    render();
    intelPost("/api/wire/feeds", { name: name, url: url }, function (j) {
      wireFeedMsg = j && j.ok ? (j.sentence || "Feed added.") : ((j && j.error) || "The feed was refused.");
      wireState = null; refreshWire(); render();
    });
  });
  mainEl.querySelectorAll("[data-wire-feed-remove]").forEach(function (b) { b.onclick = function () {
    intelPost("/api/wire/feeds/remove", { id: b.getAttribute("data-wire-feed-remove") }, function (j) {
      wireFeedMsg = j && j.ok ? "Feed removed." : ((j && j.error) || "The feed could not be removed.");
      wireState = null; refreshWire(); render();
    });
  }; });
  // Pulse (v0.5.13e, #287): the tab's acts — the same doors
  // the bridge receipts from chat.
  function pulseAct(path, body, okMsg) {
    intelPost(path, body, function (j) {
      if (j && j.pulse) pulseState = j.pulse;
      pulseMsg = j && j.ok ? okMsg : ((j && j.error) || "That pulse act did not land.");
      if (!pulseState) refreshPulse();
      render();
    });
  }
  bind("pulse-add", function () {
    var inp = document.getElementById("pulse-topic");
    var topic = inp ? inp.value.trim() : "";
    if (!topic) { pulseMsg = "Name the topic first — a watch needs something to keep a pulse on."; render(); return; }
    pulseAct("/api/pulse/create", { topic: topic }, "Watching \u201c" + topic + "\u201d — it joins the next check.");
  });
  bind("pulse-run", function () {
    intelPost("/api/pulse/run", {}, function (j) {
      if (j && j.pulse) pulseState = j.pulse;
      var n = j && j.summary ? j.summary.newTotal : 0;
      pulseMsg = j && j.ok ? ("Checked — " + (n ? n + " new item" + (n === 1 ? "" : "s") + " across the watches." : "nothing new on any watch.")) : ((j && j.error) || "The check did not run.");
      render();
    });
  });
  mainEl.querySelectorAll("[data-pulse-pause]").forEach(function (b) { b.onclick = function () { pulseAct("/api/pulse/state", { id: b.getAttribute("data-pulse-pause"), state: "paused" }, "Watch paused."); }; });
  mainEl.querySelectorAll("[data-pulse-resume]").forEach(function (b) { b.onclick = function () { pulseAct("/api/pulse/state", { id: b.getAttribute("data-pulse-resume"), state: "active" }, "Watch resumed."); }; });
  mainEl.querySelectorAll("[data-pulse-remove]").forEach(function (b) { b.onclick = function () { pulseAct("/api/pulse/remove", { id: b.getAttribute("data-pulse-remove") }, "Watch removed."); }; });
  mainEl.querySelectorAll("[data-pulse-attach]").forEach(function (b) { b.onclick = function () {
    var sel = mainEl.querySelector("[data-pulse-case='" + b.getAttribute("data-pulse-attach") + "']");
    pulseAct("/api/pulse/attach", { id: b.getAttribute("data-pulse-attach"), case: sel ? sel.value : "" }, "Case feeding set — new hits land on that case's reference shelf.");
  }; });
  bind("wire-pull", function () {
    intelPost("/api/wire/refresh", {}, function (j) {
      wireMsg = j && j.ok ? "Fetch started — the wire refreshes when the feeds answer." : ((j && j.error) || "The fetch could not start.");
      render();
      setTimeout(function () { refreshWire(); }, 2500);
    });
  });
  bind("wire-save", function () {
    var en = document.getElementById("wire-enabled");
    var iv = document.getElementById("wire-interval");
    var rt = document.getElementById("wire-retention");
    intelPost("/api/wire/settings", { enabled: en ? en.checked : true, intervalMinutes: iv ? Number(iv.value) : 30, retentionDays: rt ? Number(rt.value) : 7 }, function (j) {
      wireMsg = j && j.ok ? "Wire settings saved." : ((j && j.error) || "The settings could not be saved.");
      wireState = null; refreshWire(); render();
    });
  });
  bind("ctf-pull", function () {
    intelPost("/api/ctf/refresh", {}, function (j) {
      ctfMsg = j && j.ok ? "Fetch started — the board refreshes when CTFtime answers." : ((j && j.error) || "The fetch could not start.");
      render();
      setTimeout(function () { refreshCtf(); }, 2500);
    });
  });
  var ctfEn = document.getElementById("ctf-enabled");
  if (ctfEn) ctfEn.onchange = function () {
    intelPost("/api/ctf/settings", { enabled: ctfEn.checked }, function (j) {
      ctfMsg = j && j.ok ? "Board " + (ctfEn.checked ? "enabled." : "disabled — the board keeps its last fetch and stops polling.") : ((j && j.error) || "The setting could not be saved.");
      ctfState = null; refreshCtf(); render();
    });
  };
  bind("bounty-pull", function () {
    intelPost("/api/bounties/refresh", {}, function (j) {
      bountyMsg = j && j.ok ? "Fetch started — the board refreshes when the index answers." : ((j && j.error) || "The fetch could not start.");
      render();
      setTimeout(function () { refreshBounties(); }, 2500);
    });
  });
  var bEn = document.getElementById("bounty-enabled");
  if (bEn) bEn.onchange = function () {
    intelPost("/api/bounties/settings", { enabled: bEn.checked }, function (j) {
      bountyMsg = j && j.ok ? "Board " + (bEn.checked ? "enabled." : "disabled — the board keeps its last fetch and stops polling.") : ((j && j.error) || "The setting could not be saved.");
      bountyState = null; refreshBounties(); render();
    });
  };
}

function intelSearchHtml() {
  // v0.5.2e (#259): one search over the whole local catalog —
  // a CVE id (exact or prefix), free text, a vendor or product
  // name. The catalog's last-fetched stamp rides the results:
  // a search over a stale catalog says so on its face. Results
  // are the queue's own rows and open in the record region
  // through the same door.
  var h = "<div class='card' style='margin-top:12px'><h3 style='margin-top:0'>Search the catalog</h3>";
  h += "<div class='row' style='gap:8px'><input id='ti-search-q' placeholder='CVE id, product, vendor, or words — e.g. openssh' style='flex:1;min-height:44px' value='" + esc(intelSearch && intelSearch.q || "").replace(/"/g, "&quot;") + "'> <button class='act' id='ti-search-go' style='min-height:44px'" + (intelSearchBusy ? " disabled" : "") + ">" + (intelSearchBusy ? "Searching…" : "Search") + "</button></div>";
  h += "<div style='display:flex;flex-wrap:wrap;gap:14px;margin-top:4px;align-items:center'>";
  h += "<label style='display:flex;gap:6px;align-items:center;min-height:44px'><input type='checkbox' id='ti-search-kev'" + (intelSearch && intelSearch.kevOnly ? " checked" : "") + "> Known-exploited only</label>";
  h += "<label style='display:flex;gap:6px;align-items:center;min-height:44px'><input type='checkbox' id='ti-search-matched'" + (intelSearch && intelSearch.matchedOnly ? " checked" : "") + "> Matched to you</label>";
  h += "<label style='font-size:12px;color:var(--dim)'>Published <select id='ti-search-days' style='min-height:44px'>" + [[0, "any time"], [7, "last 7 days"], [30, "last 30 days"], [90, "last 90 days"]].map(function (p) { return "<option value='" + p[0] + "'" + (intelSearch && Number(intelSearch.days) === p[0] ? " selected" : "") + ">" + p[1] + "</option>"; }).join("") + "</select></label>";
  h += "</div>";
  if (intelSearch && intelSearch.error) h += "<p style='color:#f87171;font-size:13px;margin-top:6px'>" + esc(intelSearch.error) + "</p>";
  if (intelSearch && !intelSearch.error) {
    var st = intelSearch.stamp || {};
    var age = (st.ageHours === null || st.ageHours === undefined) ? "never fetched" : (st.ageHours < 1 ? "under an hour old" : Math.round(st.ageHours) + "h old");
    h += "<p class='mono' style='font-size:11px;color:var(--dim);margin-top:6px'>Catalog: " + (st.total || 0) + " records · last fetched " + age + " · " + esc(st.band || "never") + ((st.band === "stale" || st.band === "aging") ? " — searching a " + esc(st.band) + " catalog; Pull now narrows the gap" : "") + "</p>";
    if (!intelSearch.results.length) {
      h += "<p style='color:var(--dim);font-size:13px'>Nothing in the catalog matches — absence from the store, not proof of safety.</p>";
    } else {
      h += "<p style='font-size:12px;color:var(--dim)'>" + intelSearch.count + " match" + (intelSearch.count === 1 ? "" : "es") + (intelSearch.count > intelSearch.results.length ? " · showing " + intelSearch.results.length : "") + " — pick one and it opens in the record, like the queue.</p>";
      intelSearch.results.forEach(function (it) { h += intelItemHtml(it, it.id === intelSelId); });
    }
  }
  h += "</div>";
  return h;
}
function ticCenterHtml(scope) {
  // scope "cves" renders the CVE regions only (queue + record);
  // scope "center" renders the whole Center (rail + advisories
  // included). The awareness shell above carries the head.
  var h = "";
  if (intelMsg) h += "<p style='font-size:13px'>" + esc(intelMsg) + "</p>";
  if (CONFIG.backend !== "live") { h += "<div class='card'>The Wire works through the live bridge — the store lives in this install's config. Switch the backend to Live in Settings and it will be here.</div>"; return h; }
  if (!intelState) {
    if (!intelFetching) refreshIntel();
    return h + "<div class='card'>Gathering the picture…</div>";
  }
  var s = intelState;
  if (s.ok === false) return h + "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(s.error || "The Center is unavailable.") + "</div>";
  // Freshness strip: who collected, and how old each source is.
  var collName = (s.collector && s.collector.name) || "";
  h += "<div class='card'><div style='display:flex;flex-wrap:wrap;gap:8px;align-items:center;justify-content:space-between'><div><b>" +
    (s.mode === "receiver" ? "Receiving the digest over the Mesh" : "Collecting on this node") + "</b>" +
    (collName ? " <span style='color:var(--dim)'>· gathered by " + esc(collName) + "</span>" : "") +
    " <span style='color:var(--dim)'>· " + s.kevTotal + " known-exploited · " + s.matchedCount + " matched to you</span></div>" +
    "<button class='act' id='ti-pull' style='min-height:44px'" + (s.pulling ? " disabled" : "") + ">" + (s.pulling ? "Pulling…" : "Pull now") + "</button></div>";
  h += "<div class='tic-fresh'>";
  (s.sources || []).forEach(function (src) {
    if (src.source === "mesh" && s.mode !== "receiver") return;
    var age = src.ageHours === null || src.ageHours === undefined ? "never pulled" : (src.ageHours < 1 ? "under an hour old" : Math.round(src.ageHours) + "h old");
    h += "<span class='tic-chip'><span style='color:" + intelBandColor(src.band) + "'>●</span>&nbsp;" + esc(src.label) + " · " + esc(src.band) + " · " + age + (src.count ? " · " + src.count : "") + "</span>";
  });
  h += "</div>";
  (s.sources || []).forEach(function (src) { if (src.error) h += "<p style='color:#f87171;font-size:12px;margin-top:6px'>" + esc(src.label) + ": " + esc(src.error) + "</p>"; });
  h += "</div>";
  h += intelSearchHtml();
  // The three regions.
  h += "<div class='tic-grid'>";
  // -- triage queue
  h += "<div class='tic-queue'><div class='card'><h3 style='margin-top:0'>Triage</h3><p style='color:var(--dim);font-size:12px'>Your matches first, then what's newly exploited, then the loudest new disclosures. Ranked, never raw.</p>";
  if (!s.queue || !s.queue.length) h += "<p style='color:var(--dim)'>Nothing in the queue — either the world is quiet or the store is still empty. Pull now, or let the cadence work.</p>";
  (s.queue || []).forEach(function (it) { h += intelItemHtml(it, it.id === intelSelId); });
  if (s.movers && s.movers.length) {
    h += "<p style='margin-top:10px'><b>EPSS movers</b> <span style='color:var(--dim);font-size:12px'>— likelihood climbing:</span></p>";
    s.movers.slice(0, 6).forEach(function (m) { h += "<div class='mono' style='font-size:12px;margin-top:2px'>" + esc(m.id) + " <span style='color:#f5b544'>+" + Math.round(m.delta * 100) + " pts</span></div>"; });
  }
  h += "</div></div>";
  // -- the record
  h += "<div class='tic-detail'><div class='card'><h3 style='margin-top:0'>The record</h3>";
  if (!intelSelId) h += "<p style='color:var(--dim)'>Pick a record from the queue and it opens here — the full description, the references, and the join to your cases.</p>";
  else if (intelItemFetching && (!intelItem || intelItem.id !== intelSelId)) h += "<p style='color:var(--dim)'>Opening " + esc(intelSelId) + "…</p>";
  else if (intelItem && intelItem.id === intelSelId) {
    var it = intelItem;
    h += "<b style='color:" + caseSevColor(intelSev(it.cvss)) + ";font-size:16px'>" + esc(it.id) + "</b>";
    if (it.kev) h += " <span class='mono' style='font-size:11px;color:#f87171'>KNOWN-EXPLOITED" + (it.kevDue ? " · due " + esc(it.kevDue) : "") + (it.kevRansomware && it.kevRansomware !== "Unknown" ? " · ransomware: " + esc(it.kevRansomware) : "") + "</span>";
    if (it.matched) h += "<div style='margin-top:4px'><span class='mono' style='font-size:11px;color:#34d399'>" + (it.matched.grade === "exact" ? "MATCHES" : "mentions") + " your " + esc(it.matched.via) + " term “" + esc(it.matched.term) + "”</span></div>";
    h += "<p style='margin-top:6px'><b>" + esc(it.title || "") + "</b></p>";
    h += "<div class='mono' style='font-size:12px;color:var(--dim)'>" +
      (it.cvss !== null && it.cvss !== undefined ? "CVSS " + it.cvss + (it.cvssSeverity ? " " + esc(it.cvssSeverity) : "") + " · " : "") +
      (it.epss !== null && it.epss !== undefined ? "EPSS " + Math.round(it.epss * 100) + "% (" + Math.round((it.epssPercentile || 0) * 100) + "th pct) · " : "") +
      (it.published ? "published " + esc(String(it.published).slice(0, 10)) : "") + "</div>";
    if (it.products && it.products.length) h += "<div class='mono' style='font-size:12px;margin-top:4px'>Products: " + esc(it.products.slice(0, 8).join(", ")) + "</div>";
    if (it.description) h += "<div style='margin-top:8px;white-space:pre-wrap'>" + esc(it.description) + "</div>";
    if (it.epssHistory && it.epssHistory.length > 1) h += "<div class='mono' style='font-size:11px;color:var(--dim);margin-top:8px'>EPSS history: " + it.epssHistory.slice(0, 7).map(function (x) { return esc(x.date) + " " + Math.round(x.epss * 100) + "%"; }).join(" · ") + "</div>";
    if (it.refs && it.refs.length) h += "<div style='margin-top:8px'>" + it.refs.slice(0, 6).map(function (r) { return "<div><a href='" + esc(r) + "' target='_blank' rel='noopener' style='color:var(--cy);font-size:13px'>" + esc(r) + "</a></div>"; }).join("") + "</div>";
    if (it.exploits && it.exploits.length) h += "<div class='mono' style='font-size:12px;margin-top:8px;color:#f5b544'>Public exploit code: " + it.exploits.map(function (e) { return "EDB-" + esc(e.edb) + " (" + esc(e.type || "exploit") + " · " + esc(e.platform || "?") + ")"; }).join(" · ") + " <span style='color:var(--dim)'>— Exploit-DB, mirrored locally, joined by CVE id</span></div>";
    h += "<div class='row' style='margin-top:10px'>";
    if (openCaseState) h += "<button class='act' id='ti-file' style='min-height:44px'>File into the open case — " + esc(openCaseState.name) + "</button>";
    else h += "<span style='color:var(--dim);font-size:12px'>Open a case in Case Files and this record can be filed straight into it, citation and all.</span>";
    h += "</div>";
  } else h += "<p style='color:var(--dim)'>That record is not in the local store.</p>";
  h += "</div></div>";
  // -- the rail: brief, watchlist, collection
  if (scope !== "cves") {
  h += "<div class='tic-rail'>";
  h += "<div class='card'><h3 style='margin-top:0'>Navi's brief</h3>";
  if (s.brief) h += "<div class='mono' style='font-size:12px;white-space:pre-wrap'>" + esc(s.brief) + "</div>";
  else h += "<p style='color:var(--dim)'>No brief yet — the store is empty. Once a pull lands, this is the standing picture she carries into chat.</p>";
  h += "</div>";
  h += "<div class='card'><h3 style='margin-top:0'>Watchlist</h3><p style='color:var(--dim);font-size:12px'>The software you declare. Matching happens here, on this machine — the list is never sent to a feed.</p>";
  h += "<div style='display:flex;flex-wrap:wrap;gap:6px'>" + (s.watchlist || []).map(function (t) { return "<span class='tic-chip'>" + esc(t) + " <button class='ghost' data-ti-watch-del='" + esc(t).replace(/"/g, "&quot;") + "' style='min-height:32px;padding:0 8px' aria-label='Remove " + esc(t) + "'>×</button></span>"; }).join("") + "</div>";
  h += "<div class='row' style='margin-top:8px'><input id='ti-watch-in' placeholder='e.g. openssl, log4j, wordpress' style='flex:1;min-height:44px'> <button class='ghost' id='ti-watch-add' style='min-height:44px'>Add</button></div></div>";
  h += "<div class='card'><h3 style='margin-top:0'>Collection</h3>";
  h += "<label style='display:flex;gap:8px;align-items:center;min-height:44px'><input type='checkbox' id='ti-enabled'" + (s.enabled ? " checked" : "") + "> Center enabled</label>";
  var srcs = (s.settings && s.settings.sources) || {};
  [["nvd", "NVD (CVEs, CVSS, products)"], ["kev", "CISA KEV (known-exploited)"], ["epss", "EPSS (likelihood, daily)"], ["cisa", "CISA advisories"], ["rss", "Curated RSS (Krebs, BleepingComputer)"], ["edb", "Exploit-DB (public exploit index, daily)"]].forEach(function (pair) {
    h += "<label style='display:flex;gap:8px;align-items:center;min-height:44px'><input type='checkbox' data-ti-src='" + pair[0] + "'" + (srcs[pair[0]] !== false ? " checked" : "") + "> " + pair[1] + "</label>";
  });
  h += "<div style='margin-top:6px'><label style='font-size:12px;color:var(--dim)'>Collect every</label> <select id='ti-cadence' style='min-height:44px'>" + [3, 6, 12, 24].map(function (n) { return "<option value='" + n + "'" + (Number(s.cadenceHours) === n ? " selected" : "") + ">" + n + " hours</option>"; }).join("") + "</select></div>";
  h += "<div style='margin-top:8px'><label style='font-size:12px;color:var(--dim)'>Mesh collector (node id — blank = this node collects)</label><input id='ti-peer' value='" + esc(s.collectorPeer || "") + "' placeholder='collector node id' style='width:100%;min-height:44px;margin-top:4px'></div>";
  h += "<label style='display:flex;gap:8px;align-items:center;min-height:44px'><input type='checkbox' id='ti-fallback'" + (s.fallbackLocal ? " checked" : "") + "> If the collector is unreachable, collect locally</label>";
  h += "<p style='color:var(--dim);font-size:12px'>One collector per Mesh is the design: the digest travels, every node matches locally. A receiver never polls the feeds itself unless the fallback says so.</p>";
  h += "<div class='row' style='margin-top:8px'><button class='act' id='ti-save' style='min-height:44px'>Save collection settings</button></div></div>";
  h += "</div>";
  }
  h += "</div>";
  // -- advisories
  if (scope !== "cves" && s.advisories && s.advisories.length) {
    h += "<div class='card' style='margin-top:12px'><h3 style='margin-top:0'>Latest advisories &amp; reporting</h3>";
    s.advisories.slice(0, 10).forEach(function (a) {
      h += "<div style='margin-top:6px'><a href='" + esc(a.link) + "' target='_blank' rel='noopener' style='color:var(--cy)'>" + esc(a.title) + "</a> <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(a.source) + (a.published ? " · " + esc(String(a.published).slice(0, 16)) : "") + "</span></div>";
    });
    h += "</div>";
  }
  return h;
}
function wireThreatIntel() {
  if (!intelState && !intelFetching) refreshIntel();
  wireAwareness();
  var bind = function (id, fn) { var el = document.getElementById(id); if (el) el.onclick = fn; };
  bind("ti-pull", function () {
    intelPost("/api/intel/refresh", {}, function (j) {
      intelMsg = j && j.ok ? "Pull started — the picture refreshes when the feeds answer." : ((j && j.error) || "The pull could not start.");
      intelState = null;
      render();
      setTimeout(function () { refreshIntel(); }, 4000);
    });
  });
  mainEl.querySelectorAll("[data-ti-sel]").forEach(function (b) { b.onclick = function () { intelSelId = b.getAttribute("data-ti-sel"); intelItem = null; refreshIntelItem(intelSelId); render(); }; });
  bind("ti-file", function () {
    if (!openCaseState || !intelSelId) return;
    intelPost("/api/intel/file", { case: openCaseState.slug, cve: intelSelId }, function (j) {
      intelMsg = j && j.ok ? "Filed — " + intelSelId + " is in the case “" + openCaseState.name + "” as a finding, citation and all." : ((j && j.error) || "The filing failed.");
      render();
    });
  });
  // v0.5.2e (#259): the catalog search — one box for the whole
  // catalog, filters riding beside it; results land in
  // intelSearch and render as the queue's own rows.
  var runSearch = function () {
    var qEl = document.getElementById("ti-search-q");
    var kevEl = document.getElementById("ti-search-kev");
    var matEl = document.getElementById("ti-search-matched");
    var dayEl = document.getElementById("ti-search-days");
    var q = qEl ? qEl.value : "";
    var kev = !!(kevEl && kevEl.checked);
    var mat = !!(matEl && matEl.checked);
    var days = dayEl ? Number(dayEl.value || 0) : 0;
    intelSearchBusy = true;
    render();
    intelPost("/api/intel/search", { q: q, kevOnly: kev, matchedOnly: mat, days: days }, function (j) {
      intelSearchBusy = false;
      if (j && j.ok) intelSearch = { q: q, kevOnly: kev, matchedOnly: mat, days: days, count: j.count || 0, results: j.results || [], stamp: j.stamp || null, error: "" };
      else intelSearch = { q: q, kevOnly: kev, matchedOnly: mat, days: days, count: 0, results: [], stamp: null, error: (j && j.error) || "The search could not run." };
      render();
    });
  };
  bind("ti-search-go", runSearch);
  var sqEl = document.getElementById("ti-search-q");
  if (sqEl) sqEl.onkeydown = function (e) { if (e.key === "Enter") { e.preventDefault(); runSearch(); } };
  var saveWatch = function (terms) {
    intelPost("/api/intel/watchlist", { terms: terms }, function (j) {
      if (j && j.ok) { intelMsg = "Watchlist saved — matching recomputed against it."; intelState = null; refreshIntel(); }
      else intelMsg = (j && j.error) || "The watchlist could not be saved.";
      render();
    });
  };
  bind("ti-watch-add", function () {
    var inp = document.getElementById("ti-watch-in");
    var t = inp ? inp.value.trim().toLowerCase() : "";
    if (!t) return;
    var terms = (intelState.watchlist || []).slice();
    if (terms.indexOf(t) === -1) terms.push(t);
    saveWatch(terms);
  });
  mainEl.querySelectorAll("[data-ti-watch-del]").forEach(function (b) { b.onclick = function () { var t = b.getAttribute("data-ti-watch-del"); saveWatch((intelState.watchlist || []).filter(function (x) { return x !== t; })); }; });
  bind("ti-save", function () {
    var src = {};
    mainEl.querySelectorAll("[data-ti-src]").forEach(function (c) { src[c.getAttribute("data-ti-src")] = c.checked; });
    var peer = document.getElementById("ti-peer");
    var cad = document.getElementById("ti-cadence");
    intelPost("/api/intel/settings", {
      enabled: document.getElementById("ti-enabled").checked,
      sources: src,
      collectorPeer: peer ? peer.value.trim() : "",
      fallbackLocal: document.getElementById("ti-fallback").checked,
      cadenceHours: cad ? Number(cad.value) : 6
    }, function (j) {
      intelMsg = j && j.ok ? "Collection settings saved." : ((j && j.error) || "The settings could not be saved.");
      intelState = null;
      refreshIntel();
      render();
    });
  });
}

function caseSevColor(sev) { return { critical: "#f87171", high: "#f5b544", medium: "var(--cy)", low: "#9fb3c8", info: "var(--dim)" }[sev] || "var(--dim)"; }
function caseFilesHtml() {
  var h = "<h3>Case Files</h3>";
  h += "<p style='color:var(--dim)'>The engagement workspace. A case is a folder of plain markdown on the shared desk — <span class='mono'>cases/</span> in the Vault — so it rides the Vault's backups, opens in Obsidian as it lies, and outlives any one screen. One case at a time is <b>open</b>: the open case is where Navi files findings from chat, out loud, and where chip runs can be filed.</p>";
  if (caseMsg) h += "<p style='font-size:13px'>" + esc(caseMsg) + "</p>";
  if (CONFIG.backend !== "live") { h += "<div class='card'>Case Files work through the live bridge — the cases live in this install's Vault folder. Switch the backend to Live in Settings and they will be here.</div>"; return h; }
  if (caseDetailState) {
    var d = caseDetailState;
    var isOpen = caseOpenSlug === d.slug;
    h += "<div class='card'><button class='ghost' id='cf-back' style='min-height:44px'>← All cases</button>";
    h += "<h3 style='margin:10px 0 4px'>" + esc(d.name) + "</h3>";
    h += "<p class='mono' style='color:var(--dim);font-size:12px;margin:2px 0'>" + esc(d.slug) + " · status " + esc(d.status) + " · opened " + esc(d.opened) + "</p>";
    if (d.client) h += "<p style='color:var(--dim);margin:2px 0'><b>Client:</b> " + esc(d.client) + "</p>";
    if (d.scope) h += "<p style='color:var(--dim);margin:2px 0'><b>Scope:</b> " + esc(d.scope) + "</p>";
    h += "<div class='row cf-actions' style='margin-top:8px'><button class='ghost' id='cf-setopen' style='min-height:44px'>" + (isOpen ? "This is the open case — close it" : "Make this the open case") + "</button> <button class='act' id='cf-report' style='min-height:44px'>Export the report</button> <button class='ghost' id='cf-chat' style='min-height:44px'>Open the case chat</button></div>";
    h += "<p style='color:var(--dim);font-size:12px'>" + (isOpen ? "Navi files to this case from chat — say <i>“file that as a finding.”</i>" : "Only the open case receives filings from chat.") + "</p>";
    if (caseReportState && caseReportState.slug === d.slug) {
      h += "<div class='card' style='border-color:var(--cy)'><b>Report exported</b> — saved as <span class='mono'>report.md</span> in the case folder" + (caseReportState.withheld ? ", with <b>" + caseReportState.withheld + " internal note" + (caseReportState.withheld === 1 ? "" : "s") + " withheld and counted</b> in the document itself" : "") + ". <button class='ghost' id='cf-report-dl' style='min-height:44px'>Download report.md</button><details style='margin-top:6px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Read the report</summary><pre class='exec mono' style='margin-top:6px;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>" + esc(caseReportState.text) + "</pre></details></div>";
    }
    // Build 2 (#199): the grown-up manifest — summary first,
    // then the fields, then the edit fold that writes them.
    if (d.summary) h += "<p style='margin:6px 0'><b>Summary:</b> " + esc(d.summary) + "</p>";
    h += "<p class='mono' style='color:var(--dim);font-size:12px;margin:2px 0'>type " + esc(d.type || "investigation") + " · priority " + esc(d.priority || "normal") + (d.owner ? " · owner " + esc(d.owner) : "") + (d.tags && d.tags.length ? " · tags " + esc(d.tags.join(", ")) : "") + (d.related && d.related.length ? " · related " + esc(d.related.join(", ")) : "") + (d.status === "closed" ? " · closed " + esc(d.closed || "") + (d.outcome ? " — outcome: " + esc(d.outcome) : "") : "") + "</p>";
    h += "<div class='row' style='margin-top:6px'><label style='font-size:13px'><input type='checkbox' id='cf-sensitive-toggle'" + (d.sensitive ? " checked" : "") + "> Sensitive case — its name and summary stay off the DHD list, and Sync badges it</label></div>";
    if (caseEditMsg) h += "<p style='font-size:13px;color:" + (caseEditMsg.indexOf("could not") === -1 && caseEditMsg.indexOf("refused") === -1 ? "var(--green)" : "#f87171") + "'>" + esc(caseEditMsg) + "</p>";
    h += "<div class='row' style='margin-top:4px'><button class='ghost' id='cf-edit-toggle' style='min-height:44px'>" + (caseEditOpen ? "Close the editor" : "Edit the case") + "</button></div>";
    if (caseEditOpen) {
      h += "<div style='margin-top:8px;border-top:1px solid var(--line);padding-top:8px'>"
        + "<div class='row'><input class='grow' id='cf-e-summary' type='text' placeholder='Summary — one line' aria-label='Edit summary' value='" + esc(d.summary || "") + "'></div>"
        + "<div class='row' style='margin-top:6px'><select id='cf-e-type' aria-label='Edit type'>" + ["incident", "investigation", "research", "hunt", "locate"].map(function (t) { return "<option value='" + t + "'" + (d.type === t ? " selected" : "") + ">" + t + "</option>"; }).join("") + "</select>"
        + "<select id='cf-e-priority' aria-label='Edit priority'>" + ["low", "normal", "high", "critical"].map(function (t) { return "<option value='" + t + "'" + (d.priority === t ? " selected" : "") + ">" + t + " priority</option>"; }).join("") + "</select>"
        + "<select id='cf-e-status' aria-label='Edit status'>" + ["open", "closed"].map(function (t) { return "<option value='" + t + "'" + (d.status === t ? " selected" : "") + ">" + t + "</option>"; }).join("") + "</select></div>"
        + "<div class='row' style='margin-top:6px'><input class='grow' id='cf-e-tags' type='text' placeholder='Tags — comma separated' aria-label='Edit tags' value='" + esc((d.tags || []).join(", ")) + "'><input class='grow' id='cf-e-owner' type='text' placeholder='Owner' aria-label='Edit owner' value='" + esc(d.owner || "") + "'></div>"
        + "<div class='row' style='margin-top:6px'><input class='grow' id='cf-e-related' type='text' placeholder='Related cases — slugs, comma separated' aria-label='Edit related' value='" + esc((d.related || []).join(", ")) + "'><input class='grow' id='cf-e-outcome' type='text' placeholder='Outcome — when closing: the one-line disposition' aria-label='Edit outcome' value='" + esc(d.outcome || "") + "'></div>"
        + "<div class='row' style='margin-top:6px'><button class='act' id='cf-edit-save' style='min-height:44px'>Save the case</button></div></div>";
    }
    h += "</div>";
    if (d.unverified) {
      h += "<div class='card' style='border-color:var(--am)'><b>" + d.unverified + " unverified " + (d.unverified === 1 ? "entry awaits" : "entries await") + " review</b> — logged by Navi as leads. Promote what holds, strike what doesn't; a strike is noted in the timeline.</div>";
    }
    // Wire matches (Build 2, #199): awareness flowing in —
    // badged as MATCHES against the case's observables and
    // entities, never as findings.
    if (d.wireMatches && d.wireMatches.length) {
      h += "<div class='card'><h3>Wire matches</h3><p style='color:var(--dim);font-size:12px'>Stories and CVEs matching this case's observables or entities. A match is a lead to check — <b>not a finding</b>.</p>";
      d.wireMatches.forEach(function (w) {
        h += "<div style='margin-top:6px'><span class='mono' style='font-size:11px;color:var(--am)'>MATCH</span> <b>" + esc(w.title) + "</b><br><span style='color:var(--dim);font-size:12px'>" + esc(w.kind || "") + " · matched “" + esc(w.term) + "”" + (w.sourceLabel ? " · " + esc(w.sourceLabel) : "") + (w.kev ? " · <span style='color:#f87171'>KEV</span>" : "") + "</span>" + (w.link ? " <a href='" + esc(w.link) + "' target='_blank' rel='noreferrer' style='color:var(--cy);font-size:12px'>Open ↗</a>" : "") + "</div>";
      });
      h += "</div>";
    }
    // Rooms on this case (v0.5.7e, #273): the case face of
    // the typed link — the room's header carries it back.
    if (d.rooms && d.rooms.length) {
      h += "<div class='card'><h3>Rooms on this case</h3><p style='color:var(--dim);font-size:12px'>FediChat rooms tied to this case. The room's header carries the link back here.</p>";
      d.rooms.forEach(function (r) {
        h += "<div style='margin-top:6px'><b>" + esc(r.name) + "</b> <span style='color:var(--dim);font-size:12px'>" + esc(r.fedName) + " · " + esc(r.kind) + "</span> <button class='ghost' data-case-room='" + esc(r.fedId) + "|" + esc(r.roomId) + "' style='min-height:44px'>Open the room</button></div>";
      });
      h += "</div>";
    }
    // Findings
    h += "<div class='card'><h3>Findings</h3>";
    h += "<div class='row'><input class='grow' id='cf-f-title' type='text' placeholder='Finding title — e.g. Expired TLS certificate on the portal' aria-label='Finding title'></div>";
    h += "<div class='row' style='margin-top:6px'><select id='cf-f-sev' aria-label='Finding severity'><option value='critical'>Critical</option><option value='high'>High</option><option value='medium' selected>Medium</option><option value='low'>Low</option><option value='info'>Info</option></select><label style='color:var(--dim);font-size:13px'><input type='checkbox' id='cf-f-internal'> Internal — withheld from report exports (and counted there)</label></div>";
    h += "<textarea id='cf-f-body' rows='4' placeholder='What you found, how you proved it, what it means.' aria-label='Finding detail' style='margin-top:6px'></textarea>";
    h += "<div class='row' style='margin-top:6px'><button class='act' id='cf-f-add' style='min-height:44px'>File the finding</button>" + (lastChatExec ? " <button class='ghost' id='cf-file-exec' style='min-height:44px'>File the latest chat output as a finding</button>" : "") + "</div>";