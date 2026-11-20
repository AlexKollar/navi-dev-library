# bridge.py — part 7 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 24857–29059 of bridge.py.
Top-level symbols in this part: _chats_fold_main_ghosts, _chat_write_as, chats_converge_preview, chats_converge, _sync_now_iso, _sync_fail, _sync_auth_error, _sync_safe_rel, _sync_case_files, _json_bytes, _atomic_write, _seat_clean_image, _seat_fact_id, _seat_clean_entries, _seat_state_load, _seat_state_save, seat_state_payload, _seat_entry_tombstones, _seat_prune_tombstoned, _seat_merge, _sync_deck_collect, _sync_deck_dest, _sync_deck_merge_installs, _sync_apply_deck_card, _sync_apply_deck, _sync_doc_path, _sync_apply_docs, _sync_chat_substance, _sync_main_verdict, _sync_apply_pulse, _sync_apply_seat, sync_collect, _sync_repo, sync_fetch, _sync_reconcile_case_chats, _sync_apply_tombstones, _sync_apply_cases, sync_apply, _sync_push, _sync_record_result, _sync_finish_summary, _sync_mesh_peers, _sync_mesh_refusal_note, _sync_mesh_refusal_take, _sync_mesh_hello, _sync_mesh_peer_gate, _sync_mesh_ask, _sync_first_mesh_arrival, _purge_predates, sync_account_gate_sweep, _sync_mesh_round, _mesh_sync_foreign_account, _sync_mesh_pull, _sync_progress_set, _sync_fail_record, _SyncSuperseded, _sync_preempt_note, _sync_preempt_checkpoint, _sync_preempt_clear, _sync_superseded_result, _sync_preempt_acquire, sync_note_write, _sync_push_fire, sync_round, sync_pull_round, sync_choice_enabled, sync_set_choice, _sync_send_mesh, _sync_substance, sync_send_round, sync_receive_round, sync_override_round, sync_state_payload, _sync_trigger_pull, sync_startup_pull, _sync_note_session, sync_note_dhd_session, sync_note_app_session, _systemd_user_state, _linger_state.


def _chats_fold_main_ghosts():
    # v0.4.10e (#251): the one-time reconciliation, run at the
    # head of every convergence — a seat holding ghost mains
    # (sync-born siblings of the main seat, parked by rounds
    # that treated the main thread as one conversation among
    # many) folds them into the canonical main: messages merge
    # in by identity, nothing is lost, the ghost files go, and
    # the fold is receipted in the sync state — what folded,
    # and how many messages it carried. Ghost ids are
    # tombstoned so another seat's copy of the same ghost is
    # refused as a thread of its own (its messages, if any are
    # new, still arrive with the main seat's own record and
    # merge the same way).
    out = {"folded": 0, "messages": 0, "merged": 0}
    rows = chats_all()
    main = None
    ghosts = []
    for r in rows:
        if str(r["id"]) == MAIN_CHAT_ID and not r.get("case"):
            main = r
        elif _chat_is_main_family(r):
            ghosts.append(r)
    if main is None or not ghosts:
        return out
    ghosts.sort(key=lambda r: (r.get("updatedAt") or 0, str(r["id"])))
    receipt = []
    cur = main
    for g in ghosts:
        # v0.6.3e (#318): a ghost's own clear epoch retires its
        # pre-clear lines before the fold — the fold must never
        # resurrect content any seat emptied. (The main's own
        # epoch is enforced again at the write below.)
        merged_msgs, added = _chat_merge_messages(cur.get("messages"), _chat_clear_filter(g))
        nxt = dict(cur)
        nxt["messages"] = merged_msgs
        nxt["updatedAt"] = max(cur.get("updatedAt") or 0, g.get("updatedAt") or 0)
        saved = chat_write(nxt)
        if saved is None:
            continue
        cur = saved
        try:
            os.remove(_chat_path(str(g["id"])))
        except Exception:
            continue
        sync_record_tombstone("chats", str(g["id"]))
        receipt.append({"id": str(g["id"]), "title": str(g.get("title") or ""),
                        "messages": len(g.get("messages") or [])})
        out["folded"] += 1
        out["messages"] += len(g.get("messages") or [])
        out["merged"] += added
    if receipt:
        try:
            state = _sync_state_load()
            state["mainFold"] = {"at": time.time(), "folded": receipt,
                                 "messagesMerged": out["merged"]}
            _sync_state_save(state)
        except Exception:
            pass
    return out


def _chat_write_as(rec, target):
    # Land rec's thread under target's identity: the target
    # keeps its id, its title, and its import provenance — the
    # incoming conversation continues the copy that already
    # represents it here, instead of minting another.
    out = dict(rec)
    out["id"] = target["id"]
    out["title"] = target.get("title") or rec.get("title") or "Chat"
    if target.get("fromSyncOf"):
        out["fromSyncOf"] = target["fromSyncOf"]
    else:
        out.pop("fromSyncOf", None)
    return chat_write(out)


def chats_converge_preview(incoming):
    summary = {"added": 0, "replaced": 0, "kept": 0, "siblings": 0, "mainMerged": 0}
    rows = chats_all()
    local, by_conv = _chat_local_maps(rows)
    held_keys = {_chat_msgs_key(r) for r in rows}
    for raw in (incoming or []):
        rec = _chat_normalize(raw)
        if rec is None:
            continue
        if _chat_is_main_family(rec):
            summary["mainMerged"] += 1  # v0.4.10e (#251): mains merge, never park
            continue
        cur = local.get(str(rec["id"]))
        if cur is not None and _chat_conv_key(cur) != _chat_conv_key(rec):
            cur = None  # same id, different conversation (two seats' mains)
        if cur is None:
            rep = by_conv.get(_chat_conv_key(rec))
            if rep is not None and str(rep["id"]) != str(rec["id"]):
                cur = rep
        if cur is None and str(rec["id"]) not in local:
            summary["added"] += 1
        elif cur is None:
            holder = local.get(str(rec["id"]))
            if holder is not None and _chat_msgs_prefix(rec, holder):
                summary["kept"] += 1
            else:
                summary["siblings"] += 1  # the id is held by another conversation
        else:
            v = _chat_merge_verdict(cur, rec)
            if v == "sibling" and _chat_msgs_key(rec) in held_keys:
                v = "kept"
            summary["replaced" if v == "replaced" else "siblings" if v == "sibling" else "kept"] += 1
    return summary


def chats_converge(incoming, incoming_uploads=None):
    # Merging a FOREIGN copy (backup restore / badge sync) into
    # this store — see the section header for the rules. Returns a
    # summary the caller can speak out loud.
    # Build 5 (#218): identity is by CONVERSATION (origin id +
    # createdAt — see _chat_conv_key), not by chat id alone and
    # never by import event. A foreign copy whose id is held by a
    # different conversation (two seats' main chats) is judged
    # against its own conversation's representative here; a fork
    # parks under the deterministic _chat_sibling_id; and the
    # store first dedupes the timestamp-id copies older builds
    # minted of a single conversation.
    fold = _chats_fold_main_ghosts()
    summary = {"added": 0, "replaced": 0, "kept": 0, "siblings": 0, "uploads": 0,
               "deduped": _chats_dedupe_imports(),
               "mainFolded": fold["folded"], "mainFoldedMessages": fold["merged"],
               "mainMerged": 0, "mainMergedMessages": 0}
    rows = chats_all()
    local, by_conv = _chat_local_maps(rows)
    held_keys = {_chat_msgs_key(r) for r in rows}
    day = time.strftime("%Y-%m-%d", time.localtime())
    for raw in (incoming or []):
        rec = _chat_normalize(raw)
        if rec is None:
            continue
        if _chat_is_main_family(rec):
            # v0.4.10e (#251): a record of the main seat merges
            # INTO the canonical main — the union of the two
            # copies, each message exactly once — and never
            # parks as a sibling, never lands as another
            # thread. A seat holding no main yet adopts the
            # record AS its main.
            cur_main = local.get(MAIN_CHAT_ID)
            if cur_main is not None and cur_main.get("case"):
                cur_main = None
            if cur_main is None:
                adopt = dict(rec)
                adopt["id"] = MAIN_CHAT_ID
                adopt.pop("fromSyncOf", None)
                saved = chat_write(adopt)
                if saved is not None:
                    local[MAIN_CHAT_ID] = saved
                    by_conv.setdefault(_chat_conv_key(saved), saved)
                    held_keys.add(_chat_msgs_key(saved))
            else:
                merged_msgs, _added = _chat_merge_messages(
                    cur_main.get("messages"), rec.get("messages"))
                summary["mainMergedMessages"] += _added
                nxt = dict(cur_main)
                nxt["messages"] = merged_msgs
                nxt["updatedAt"] = max(cur_main.get("updatedAt") or 0,
                                       rec.get("updatedAt") or 0)
                # v0.6.3e (#318): the clear epoch converges
                # with the thread — the newer clear governs the
                # merged record, whichever seat spoke it, and
                # its pre-clear lines yield here (and again at
                # the store door, where chat_write enforces it).
                _epochs = [r.get("clearedAt") or 0 for r in (cur_main, rec)]
                if max(_epochs) > 0:
                    _winner = cur_main if _epochs[0] >= _epochs[1] else rec
                    nxt["clearedAt"] = max(_epochs)
                    if _winner.get("clearedBy"):
                        nxt["clearedBy"] = _winner["clearedBy"]
                    nxt["messages"] = _chat_clear_filter(nxt)
                    summary["mainCleared"] = summary.get("mainCleared", 0) + 1
                saved = chat_write(nxt)
                if saved is not None:
                    local[MAIN_CHAT_ID] = saved
                    held_keys.add(_chat_msgs_key(saved))
            summary["mainMerged"] += 1
            continue
        key = _chat_conv_key(rec)
        cur = local.get(str(rec["id"]))
        if cur is not None and _chat_conv_key(cur) != key:
            # The id is taken by a DIFFERENT conversation (the
            # standing case: two seats' main chats, both id "1").
            # This record is judged against its own conversation's
            # representative below, never against the id's holder.
            cur = None
        if cur is None:
            rep = by_conv.get(key)
            if rep is not None and str(rep["id"]) != str(rec["id"]):
                # This conversation already has a representative
                # here — an imported copy minted by an earlier
                # round, under whatever id that round gave it.
                # Converge into it: identity by conversation,
                # not by import event (Build 5, #218).
                cur = rep
        if cur is None and str(rec["id"]) not in local:
            saved = chat_write(rec)
            if saved is not None:
                local[str(saved["id"])] = saved
                by_conv.setdefault(_chat_conv_key(saved), saved)
                held_keys.add(_chat_msgs_key(saved))
            summary["added"] += 1
            continue
        if cur is None:
            # No representative, but the id itself is held by a
            # different conversation — the record parks as this
            # conversation's sibling below; it never writes
            # under the held id. One exception: a record the
            # holder already contains line for line (an older
            # generation of the holder's own thread, arriving
            # under a rewritten birth stamp) is simply kept —
            # parking it would duplicate content the holder
            # already has.
            holder = local.get(str(rec["id"]))
            if holder is not None and _chat_msgs_prefix(rec, holder):
                summary["kept"] += 1
                continue
            verdict = "sibling"
        else:
            verdict = _chat_merge_verdict(cur, rec)
        if verdict == "sibling" and _chat_msgs_key(rec) in held_keys:
            # The same diverged copy was already converged once and
            # its sibling is on file — a repeated sync of the same
            # backup must not mint a fresh sibling every time.
            verdict = "kept"
        if verdict == "replaced":
            if str(cur["id"]) == str(rec["id"]):
                saved = chat_write(rec)
            else:
                saved = _chat_write_as(rec, cur)
            if saved is not None:
                local[str(saved["id"])] = saved
                held_keys.add(_chat_msgs_key(saved))
            summary["replaced"] += 1
        elif verdict == "kept":
            summary["kept"] += 1
        else:
            # A genuine fork parks as a sibling — under the
            # conversation's deterministic import id (Build 5,
            # #218), so the next round recognizes this exact
            # copy instead of minting another.
            sib = dict(rec)
            base = _chat_sibling_id(rec)
            nid = base
            n = 2
            while os.path.exists(_chat_path(nid)):
                holder = local.get(nid)
                if holder is not None and _chat_msgs_key(holder) == _chat_msgs_key(rec):
                    break  # this exact copy is already parked here
                nid = base + "-" + str(n)
                n += 1
            sib["id"] = nid
            sib["fromSyncOf"] = key[0]
            sib["title"] = (str(rec.get("title") or "Chat").split(" (from sync ")[0] + " (from sync " + day + ")")[:200]
            saved = chat_write(sib)
            if saved is not None:
                local[str(saved["id"])] = saved
                held_keys.add(_chat_msgs_key(saved))
            summary["siblings"] += 1
    if incoming_uploads is not None:
        def _up_key(r):
            return str(r.get("id") or (str(r.get("name") or "") + "|" + str(r.get("at") or "")))
        merged = {_up_key(r): r for r in chats_uploads()}
        for r in incoming_uploads:
            if not isinstance(r, dict):
                continue
            k = _up_key(r)
            if k not in merged:
                merged[k] = r
                summary["uploads"] += 1
        if summary["uploads"]:
            chats_save_uploads(list(merged.values()))
    summary["changed"] = summary["added"] + summary["replaced"] + summary["siblings"] + summary["uploads"] + summary["deduped"] + summary["mainFolded"] + summary["mainMerged"]
    return summary


# --- Quick Sync (0.2.0e Build 3, issue #176) -------------------------
# The Operator Badge backup round (#136) is proven and deliberate;
# Quick Sync is its fluid sibling, ruled by Alex the night the
# badge round passed in the field (2026-11-13): a Sync control in
# Apex | Utilities that opens nothing and pops a card — one tap
# runs the whole round (pull, converge, push) so work started on
# the laptop is pickable-up on the phone over DHD, and vice versa.
# Build 6 adds the second transport (phase 2): the Mesh carries
# the round whenever a paired peer's road is lit — see the sync
# handlers beside the Mesh server and _sync_mesh_round below;
# the repo remains the fallback and the convergence semantics
# are shared by both.
#
# The sync set is the living work: the chat store AND Case Files.
# Settings, Soul/Memory, chips and RAG packs stay with the full
# badge backup (chips and packs keep themselves synced through
# their own channels anyway). The transport is the backup repo,
# under a sync/ prefix the restore format never touches, riding
# the OPERATOR identity — per #173's separation, never the
# read-only Runtime PAT. The badge in its slot is the round's
# identity root and trigger, as #136 ruled: a pointer, never an
# authenticator; the Operator sign-in remains the only key, and a
# signed-out round is refused and NAMED, never silently skipped.
#
# Convergence is the badge round's semantics (chats_converge)
# plus the tombstones of the section above: deletions travel as
# records and yield only to genuinely newer work. Diverged chats
# park as dated siblings; a case file that diverged inside the
# same second parks its remote copy as a ".from-sync-<date>"
# sibling file — nothing is ever silently lost, and only a parked
# pair makes the card say "Needs your eyes", naming it.
#
# Pull triggers: the button (the full round), app start, and DHD
# session start (pull + converge each). No polling daemon; the
# Mesh peer-contact trigger is phase 2, with the Mesh transport.

SYNC_FORMAT = "navi-sync/2"
SYNC_PREFIX = "sync/"
_SYNC_LOCK = threading.Lock()
_SYNC_DHD_SEEN = {}
_SYNC_DHD_LOCK = threading.Lock()
_SYNC_DHD_SESSION_GAP = 300
_SYNC_TRIGGER_DEBOUNCE = 60


def _sync_now_iso():
    return _now_iso()


def _sync_fail(status, step):
    # Build 2's naming discipline (#173), applied to sync: every
    # failure the card can show is a classified sentence — held
    # offline, named auth, named rate with its wait.
    if status == 0:
        return {"kind": "offline", "state": "held",
                "error": "Can't reach GitHub — " + step + " did not happen. The round is held; the next trigger retries it."}
    if status in (401, 403, 429):
        rate = github_rate_state() or {}
        core = rate.get("core") if isinstance(rate, dict) else None
        spent = status == 429 or (isinstance(core, dict)
                                  and (core.get("remaining") or 0) == 0
                                  and (core.get("reset") or 0) > time.time())
        if spent:
            wait = _gh_reset_phrase(core.get("reset")) if isinstance(core, dict) else ""
            return {"kind": "rate", "state": "held",
                    "error": "GitHub's rate limit is spent." + wait + " The round is held until the bucket refills — nothing was half-written."}
        return {"kind": "auth", "state": "auth",
                "error": "GitHub refused the Operator credential (HTTP " + str(status) + ") while sync tried to " + step + " — the sign-in may have expired or been revoked. Sign in again in Settings → GitHub & Backup."}
    return {"kind": "error", "state": "held",
            "error": "GitHub answered HTTP " + str(status) + " while sync tried to " + step + ". The round is held; this step wrote nothing."}


def _sync_auth_error():
    return {"ok": False, "state": "auth", "kind": "auth", "route": "Backup repo",
            "error": "Not signed in to GitHub — sync is anchored to your GitHub account, and this machine isn't signed in. Sign in with the account your other machines use (Settings → GitHub & Backup) and the round carries chats, Case Files, the ChipDeck, her Soul and memory book, and your seat under it. Nothing was sent anywhere."}


def _sync_safe_rel(rel):
    # A remote-supplied relative path is jailed by construction:
    # no empty, dot, or parent parts, no backslashes, no NUL.
    parts = str(rel or "").split("/")
    if not parts:
        return None
    for p in parts:
        if p in ("", ".", "..") or "\\" in p or "\x00" in p:
            return None
    return "/".join(parts)


def _sync_case_files(slug):
    # {rel: {"sha": hex, "mtime": epoch}} for one case folder, or
    # None when the case is not on this machine.
    d = _case_dir(slug)
    if not d or not os.path.isdir(d):
        return None
    out = {}
    for base, dirs, names in os.walk(d):
        dirs.sort()
        for name in sorted(names):
            full = os.path.join(base, name)
            if not os.path.isfile(full):
                continue
            rel = os.path.relpath(full, d).replace(os.sep, "/")
            try:
                with open(full, "rb") as f:
                    data = f.read()
                out[rel] = {"sha": hashlib.sha256(data).hexdigest(),
                            "mtime": os.path.getmtime(full)}
            except Exception:
                continue
    return out


# ---------------------------------------------------------------
# Seat state (v0.3.0e Build 1, #200): what has always lived in the
# PAGE's local storage rather than on disk — her memory entries,
# the operator's picture and badge. The Sync Update carries them
# in the sync set (sync/seat.json), anchored to the account like
# everything else, so the conversation — and the face that
# started it — can be picked up on any screen. The bridge keeps
# the canonical merged copy here; a page pushes its state and
# adopts the converged answer.
SEAT_STATE_PATH = os.path.join(ROOT, "config", "seat-state.json")


def _json_bytes(obj):
    return json.dumps(obj, indent=2).encode()


def _atomic_write(path, data):
    # Write-then-rename, the pattern the codebase uses at every
    # state file: a reader never sees a half-written file.
    d = os.path.dirname(path)
    if d:
        os.makedirs(d, exist_ok=True)
    tmp = path + ".tmp"
    with open(tmp, "wb") as f:
        f.write(data)
    os.replace(tmp, path)
_SEAT_LOCK = threading.Lock()


def _seat_clean_image(value):
    # The same discipline the dashboard image fields answer to:
    # a data-URL string under the cap, else "".
    if not isinstance(value, str):
        return ""
    return _dashboard_image_field(value)


def _seat_fact_id(text):
    # v0.5.1e (#256): a memory entry's identity for convergence.
    # memory_fact_key names the semantic facts (name / home /
    # work / favorite:*) — two texts under one semantic key are
    # the same fact, corrected. Everything else is its own fact,
    # identified by its normalized text (the page's seatFactKey
    # rule). Through v0.4.10e the seat machinery used
    # memory_fact_key ALONE as the dedupe key: it returns None
    # for ordinary facts, so every ordinary entry shared one
    # key — the cleaner kept only the first ordinary entry and
    # the merge refused the rest. That is the convicted heart
    # of the operator-class gap: the road existed, and the
    # identity function broke it.
    k = memory_fact_key(text)
    if k:
        return "fact:" + k
    return "text:" + re.sub(r"\s+", " ", str(text or "").lower()).strip()


_SEAT_ORIGINS = ("manual", "trigger", "autonomous", "correction",
                 "auto", "seed", "sync")


def _seat_clean_entries(rows):
    out = []
    seen = set()
    for row in (rows if isinstance(rows, list) else []):
        if not isinstance(row, dict):
            continue
        t = str(row.get("t") or "").strip()
        if not t:
            continue
        fid = _seat_fact_id(t)
        if fid in seen:
            continue
        seen.add(fid)
        o = str(row.get("o") or "manual")
        if o not in _SEAT_ORIGINS:
            o = "manual"
        out.append({"t": t[:600], "o": o})
    return out


def _seat_state_load():
    try:
        with open(SEAT_STATE_PATH, "r", encoding="utf-8") as fh:
            raw = json.load(fh)
    except Exception:
        raw = {}
    if not isinstance(raw, dict):
        raw = {}
    try:
        picture_at = float(raw.get("pictureAt") or 0)
    except Exception:
        picture_at = 0
    try:
        badge_at = float(raw.get("badgeAt") or 0)
    except Exception:
        badge_at = 0
    try:
        name_at = float(raw.get("nameAt") or 0)
    except Exception:
        name_at = 0
    return {
        "entries": _seat_clean_entries(raw.get("entries")),
        "operatorPicture": _seat_clean_image(raw.get("operatorPicture")),
        "operatorBadge": _seat_clean_image(raw.get("operatorBadge")),
        "pictureAt": picture_at,
        "badgeAt": badge_at,
        "operatorName": str(raw.get("operatorName") or ""),
        "nameAt": name_at,
    }


def _seat_state_save(state):
    try:
        _atomic_write(SEAT_STATE_PATH, _json_bytes(state))
    except Exception:
        pass


def seat_state_payload():
    return _seat_state_load()


def _seat_entry_tombstones():
    # The forgotten facts, as {fact id: epoch} — the "entries"
    # kind of the sync tombstone store (#256, on #253's model:
    # a tombstone is a fact at any timestamp, pruned only by
    # the standing 90-day rule).
    try:
        state = _sync_state_load()
        tombs = _sync_tombstones_pruned(state.get("tombstones") or {})
        rows = tombs.get("entries") or {}
        return {str(k): float(v) for k, v in rows.items()
                if isinstance(v, (int, float))}
    except Exception:
        return {}


def _seat_prune_tombstoned(st, tombs=None):
    # Drop the stored entries a tombstone covers; returns the
    # count. Runs on every merge, so a round carrying only a
    # tombstone still lands the forget on this seat's store.
    if tombs is None:
        tombs = _seat_entry_tombstones()
    if not tombs:
        return 0
    kept = [r for r in st["entries"] if _seat_fact_id(r["t"]) not in tombs]
    gone = len(st["entries"]) - len(kept)
    st["entries"] = kept
    return gone


def _seat_merge(incoming, source="page"):
    # Merge a page's (or a sync set's) seat state into the stored
    # one — the operator class (#256): her memory entries about
    # the operator, his name, his picture and his badge,
    # converging as one identifiable class, with the bridge's
    # copy the truth every page reads.
    # Entries union by fact identity (_seat_fact_id) and keep
    # their origins end-to-end — an arrival is never re-stamped
    # on the road. A same-fact arrival whose text differs is a
    # correction: its text wins. A forgotten fact is a
    # tombstone: stored copies are pruned and arrivals it
    # covers are refused, however fresh the offering seat's
    # copy claims to be. The name, picture and badge are single
    # values: last-writer-wins by the sender's stamp, and a
    # first fill (stored empty) always lands.
    # Returns (state, changes) with the movement counted.
    if not isinstance(incoming, dict):
        incoming = {}
    with _SEAT_LOCK:
        st = _seat_state_load()
        changes = {"entriesAdded": 0, "entriesRemoved": 0,
                   "pictureChanged": False, "badgeChanged": False,
                   "nameChanged": False}
        # The forgets this offering carries land first, so an
        # entry offered and forgotten in the same breath is
        # judged by the forget.
        for row in (incoming.get("forgotten") or []):
            t = row.get("t") if isinstance(row, dict) else row
            t = str(t or "").strip()
            if t:
                sync_record_tombstone("entries", _seat_fact_id(t))
        tombs = _seat_entry_tombstones()
        changes["entriesRemoved"] += _seat_prune_tombstoned(st, tombs)
        have = {}
        for i, r in enumerate(st["entries"]):
            have.setdefault(_seat_fact_id(r["t"]), i)
        for row in _seat_clean_entries(incoming.get("entries")):
            fid = _seat_fact_id(row["t"])
            if fid in tombs:
                continue
            if fid not in have:
                st["entries"].append({"t": row["t"], "o": row["o"]})
                have[fid] = len(st["entries"]) - 1
                changes["entriesAdded"] += 1
            else:
                cur = st["entries"][have[fid]]
                if cur["t"] != row["t"]:
                    cur["t"] = row["t"]
                    if row["o"] == "correction":
                        cur["o"] = "correction"
        for field, stamp_key, flag in (
                ("operatorPicture", "pictureAt", "pictureChanged"),
                ("operatorBadge", "badgeAt", "badgeChanged")):
            val = _seat_clean_image(incoming.get(field))
            if not val or val == st[field]:
                continue
            try:
                stamp = float(incoming.get(stamp_key) or 0)
            except Exception:
                stamp = 0
            if not st[field] or stamp >= st[stamp_key]:
                st[field] = val
                st[stamp_key] = stamp or time.time()
                changes[flag] = True
        name = str(incoming.get("operatorName") or "")
        if name and name != st["operatorName"]:
            try:
                stamp = float(incoming.get("nameAt") or 0)
            except Exception:
                stamp = 0
            if not st["operatorName"] or stamp >= st["nameAt"]:
                st["operatorName"] = name
                st["nameAt"] = stamp or time.time()
                changes["nameChanged"] = True
        _seat_state_save(st)
        if changes["nameChanged"]:
            # The bridge's own config follows the converged
            # name, so her persona greets the operator by the
            # name every seat agreed on.
            try:
                cfg = load_config()
                if str(cfg.get("operator_name") or "") != st["operatorName"]:
                    cfg["operator_name"] = st["operatorName"]
                    save_config(cfg)
            except Exception:
                pass
        if source == "page" and (changes["entriesAdded"]
                                 or changes["entriesRemoved"]
                                 or changes["pictureChanged"]
                                 or changes["badgeChanged"]
                                 or changes["nameChanged"]):
            sync_note_write("seat")
        return st, changes


def _sync_deck_collect(cfg):
    # The ChipDeck as ONE sync unit (v0.3.0e Build 1, #200): the
    # chips (cartridges + their install record), the RAG packs
    # (files + their record), and the cards that tie machines
    # together — this node's own pairing card, one re-drawn per
    # paired peer, and a gateway card for every fedinet this node
    # keeps. Cards carry identity and public halves only, exactly
    # as they do in the backup; key material never rides.
    files = {}
    index = {}

    def put(rel, data, mtime):
        files[SYNC_PREFIX + "deck/" + rel] = data
        index[rel] = {"sha": hashlib.sha256(data).hexdigest(),
                      "size": len(data), "mtime": int(mtime)}

    try:
        raw = read_file(CHIP_INSTALLS_PATH)
        if raw:
            put("chips/installed.json", raw.encode(),
                os.path.getmtime(CHIP_INSTALLS_PATH))
    except Exception:
        pass
    for row in load_chip_installs():
        local_file = row.get("local_file") if isinstance(row, dict) else None
        if not local_file:
            continue
        base = os.path.basename(str(local_file))
        path = os.path.join(CHIPS_DIR, base)
        if os.path.isfile(path):
            try:
                with open(path, "rb") as f:
                    put("chips/" + base, f.read(), os.path.getmtime(path))
            except Exception:
                continue
    try:
        raw = read_file(RAG_INSTALLS_PATH)
        if raw:
            put("rag-packs/installed.json", raw.encode(),
                os.path.getmtime(RAG_INSTALLS_PATH))
    except Exception:
        pass
    if os.path.isdir(RAG_PACKS_DIR):
        for slug in sorted(os.listdir(RAG_PACKS_DIR)):
            pdir = os.path.join(RAG_PACKS_DIR, slug)
            if not os.path.isdir(pdir):
                continue
            for name in sorted(os.listdir(pdir)):
                fpath = os.path.join(pdir, name)
                if os.path.isfile(fpath):
                    try:
                        with open(fpath, "rb") as f:
                            put("rag-packs/" + slug + "/" + name,
                                f.read(), os.path.getmtime(fpath))
                    except Exception:
                        continue
    now = time.time()
    try:
        png = build_mesh_card_png(cfg)
        if png:
            put("cards/own.png", png, now)
    except Exception:
        pass
    for peer in mesh_settings(cfg)["borrow"]["peers"]:
        try:
            if not isinstance(peer, dict):
                continue
            png = build_mesh_peer_card_png(peer)
            nid = re.sub(r"[^A-Za-z0-9_-]", "", str(peer.get("nodeId") or ""))[:12]
            if png and nid:
                put("cards/peer-" + nid + ".png", png, now)
        except Exception:
            continue
    for fed in federation_settings(cfg):
        try:
            # v0.6.4e (#321): a departed fedinet's card is never
            # dealt again — even if a ghost copy of the record
            # stands in this node's settings, the tombstone
            # outranks it, so the ghost cannot spread by deck.
            if _fed_departed(fed.get("id")):
                continue
            png = build_gateway_card_png(cfg, fed)
            fid = re.sub(r"[^A-Za-z0-9_-]", "", str(fed.get("id") or ""))[:16]
            if png and fid:
                put("cards/gateway-" + fid + ".png", png, now)
        except Exception:
            continue
    return files, index


def _sync_deck_dest(rel):
    # The landing path for a deck FILE, or None when the name is
    # not one the deck deals in. Cards and install records are
    # handled by their own paths and never land here.
    parts = rel.split("/")
    if len(parts) == 2 and parts[0] == "chips" and parts[1] not in ("", "installed.json"):
        return os.path.join(CHIPS_DIR, os.path.basename(parts[1]))
    if len(parts) == 3 and parts[0] == "rag-packs":
        if not re.fullmatch(r"[A-Za-z0-9][A-Za-z0-9._-]*", parts[1]):
            return None
        return os.path.join(RAG_PACKS_DIR, parts[1], os.path.basename(parts[2]))
    return None


def _sync_deck_merge_installs(rel, data, summary):
    # installed.json records union-merge: a record this machine
    # lacks is adopted whole; a record it holds keeps the local
    # row (install facts are machine-local). Returns True when
    # the local record file changed.
    if rel == "chips/installed.json":
        path, loader = CHIP_INSTALLS_PATH, load_chip_installs
        keyf = lambda r: str(r.get("local_file") or r.get("name") or "")
    elif rel == "rag-packs/installed.json":
        path, loader = RAG_INSTALLS_PATH, load_rag_installs
        keyf = lambda r: str(r.get("slug") or r.get("name") or "")
    else:
        return False
    try:
        remote = json.loads(data.decode("utf-8", "replace"))
    except Exception:
        return False
    if not isinstance(remote, list):
        return False
    local = loader()
    have = {keyf(r) for r in local if isinstance(r, dict)}
    merged = list(local)
    added = 0
    for row in remote:
        if not isinstance(row, dict):
            continue
        k = keyf(row)
        if k and k not in have:
            merged.append(row)
            have.add(k)
            added += 1
    if added:
        try:
            _atomic_write(path, _json_bytes(merged))
            summary["deckFilesPulled"] += 1
            return True
        except Exception:
            return False
    return False


def _sync_apply_deck_card(cfg, name, data, summary, before_peers, before_feds):
    # A card in the deck is not a file to copy — it is an act to
    # perform. A pairing card pairs, idempotently, through the
    # same validation an operator's own import walks; a gateway
    # card admits this node to a fedinet it does not keep yet —
    # admission and introduction only, never trust (Build 8's
    # law), and never a fedinet already kept.
    if name.startswith("gateway-"):
        payload, _err = parse_gateway_card_png(data)
        if not payload:
            return
        fed = payload.get("fedinet") if isinstance(payload.get("fedinet"), dict) else {}
        fid = str(payload.get("fedinetId") or fed.get("id") or "")
        if not fid or fid in before_feds:
            return
        if _fed_departed(fid):
            # v0.6.4e (#321): refused BY NAME in the round's
            # summary — a departed fedinet's card does not
            # re-admit, and the refusal is a stated fact of
            # the round, not a silent skip. Only a manual
            # import at the Federation door clears it.
            summary.setdefault("ghostRefusals", []).append(
                str(payload.get("name") or fed.get("name") or fid))
            return
        got, err = _fed_import_gateway_payload(cfg, payload, trust_default=False)
        if got and not err:
            before_feds.add(fid)
            summary["cardsImported"] += 1
        return
    peer, _err = parse_mesh_card_png(data)
    if not peer:
        return
    nid = str(peer.get("nodeId") or "")
    if not nid or nid == mesh_node_id():
        return
    if mesh_add_peer(cfg, peer) == "" and nid not in before_peers:
        before_peers.add(nid)
        summary["cardsImported"] += 1


def _sync_apply_deck(cfg, fetched, summary):
    files = fetched["files"]
    raw = files.get(SYNC_PREFIX + "deck-index.json")
    if raw is None:
        return
    try:
        index = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return
    if not isinstance(index, dict):
        return
    before_peers = {str(p.get("nodeId") or "")
                    for p in mesh_settings(cfg)["borrow"]["peers"]
                    if isinstance(p, dict)}
    before_feds = {str(f.get("id") or "") for f in federation_settings(cfg)}
    gateway_cards = []
    for rel in sorted(index):
        ent = index[rel]
        if not isinstance(ent, dict):
            continue
        data = files.get(SYNC_PREFIX + "deck/" + rel)
        if data is None:
            continue
        if rel.startswith("cards/"):
            if rel.startswith("cards/gateway-"):
                # Gateway cards land LAST, each on a freshly
                # loaded config: the import saves the config
                # itself, and a pairing add working from this
                # round's earlier snapshot must not be able to
                # save over the admission (and vice versa).
                gateway_cards.append((rel.split("/", 1)[1], data))
                continue
            _sync_apply_deck_card(cfg, rel.split("/", 1)[1], data,
                                  summary, before_peers, before_feds)
            continue
        if rel in ("chips/installed.json", "rag-packs/installed.json"):
            _sync_deck_merge_installs(rel, data, summary)
            continue
        dest = _sync_deck_dest(rel)
        if dest is None:
            continue
        remote_sha = str(ent.get("sha") or "")
        try:
            remote_mtime = int(ent.get("mtime") or 0)
        except Exception:
            remote_mtime = 0
        if os.path.isfile(dest):
            try:
                with open(dest, "rb") as f:
                    local_raw = f.read()
                local_mtime = int(os.path.getmtime(dest))
            except Exception:
                continue
            if hashlib.sha256(local_raw).hexdigest() == remote_sha:
                continue
            if local_mtime > remote_mtime:
                continue
            if local_mtime == remote_mtime:
                # Same-second divergence: the arriving copy parks
                # beside the local one, as a case file would.
                stamp = time.strftime("%Y-%m-%d", time.localtime(remote_mtime or time.time()))
                parked = dest + ".from-sync-" + stamp
                n = 2
                while os.path.exists(parked):
                    parked = dest + ".from-sync-" + stamp + "-" + str(n)
                    n += 1
                try:
                    _atomic_write(parked, data)
                except Exception:
                    pass
                if "the ChipDeck" not in summary["eyes"]:
                    summary["eyes"].append("the ChipDeck")
                continue
        try:
            os.makedirs(os.path.dirname(dest), exist_ok=True)
            _atomic_write(dest, data)
            summary["deckFilesPulled"] += 1
        except Exception:
            continue
    for name, data in gateway_cards:
        _sync_apply_deck_card(load_config(), name, data,
                              summary, before_peers, before_feds)


_SYNC_DOCS = (("soul", "Soul"), ("memory", "memory book"))


def _sync_doc_path(name):
    return SOUL_PATH if name == "soul" else MEMORY_PATH


def _sync_apply_docs(fetched, summary, state):
    # Her Soul and her memory book converge per document — never
    # a silent overwrite (v0.3.0e Build 1, #200). Same bytes:
    # nothing. One side untouched since the last sync (its hash
    # is the recorded one): the other side's text lands. Both
    # changed: the newer text wins, the losing text is parked
    # beside the document as a named conflict copy, and the
    # document is named for the operator's eyes.
    files = fetched["files"]
    raw = files.get(SYNC_PREFIX + "docs-index.json")
    if raw is None:
        return
    try:
        index = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return
    if not isinstance(index, dict):
        return
    doc_sync = state.get("docSync")
    if not isinstance(doc_sync, dict):
        doc_sync = {}
    for name, label in _SYNC_DOCS:
        data = files.get(SYNC_PREFIX + "docs/" + name + ".md")
        ent = index.get(name)
        if data is None or not isinstance(ent, dict):
            continue
        path = _sync_doc_path(name)
        remote_sha = hashlib.sha256(data).hexdigest()
        local_raw = b""
        local_mtime = 0
        if os.path.isfile(path):
            try:
                with open(path, "rb") as f:
                    local_raw = f.read()
                local_mtime = int(os.path.getmtime(path))
            except Exception:
                local_raw, local_mtime = b"", 0
        local_sha = hashlib.sha256(local_raw).hexdigest() if local_raw else ""
        last = str(doc_sync.get(name) or "")
        if remote_sha == local_sha:
            doc_sync[name] = remote_sha
            continue
        if local_sha == last:
            try:
                _atomic_write(path, data)
                doc_sync[name] = remote_sha
                summary["docsPulled"] += 1
            except Exception:
                pass
            continue
        if remote_sha == last:
            continue
        try:
            remote_mtime = int(ent.get("mtime") or 0)
        except Exception:
            remote_mtime = 0
        stamp = time.strftime("%Y-%m-%d", time.localtime())
        base = os.path.splitext(path)[0] + ".from-sync-" + stamp
        parked = base + ".md"
        n = 2
        while os.path.exists(parked):
            parked = base + "-" + str(n) + ".md"
            n += 1
        if remote_mtime >= local_mtime:
            loser = local_raw
            try:
                _atomic_write(path, data)
                doc_sync[name] = remote_sha
                summary["docsPulled"] += 1
            except Exception:
                continue
        else:
            loser = data
            doc_sync[name] = local_sha
        try:
            _atomic_write(parked, loser)
        except Exception:
            pass
        summary["eyes"].append("her " + label)
    state["docSync"] = doc_sync


def _sync_chat_substance(chat):
    # v0.5.13e (#262): a chat's substance, in the two numbers
    # the sync law verifies by — its message count and a digest
    # of the messages themselves. The digest is over the
    # canonical message list only, so envelope drift (titles,
    # stamps) never reads as lost substance.
    msgs = (chat or {}).get("messages") if isinstance(chat, dict) else None
    if not isinstance(msgs, list):
        msgs = []
    canon = json.dumps(msgs, sort_keys=True, default=str).encode("utf-8")
    return len(msgs), hashlib.sha256(canon).hexdigest()


def _sync_main_verdict(fetched):
    # v0.6.3e (#318): the post-round substance check on the main
    # thread. The converged main is a pure function of the two
    # copies and the clear epoch: the union of both sides'
    # messages, less whatever the newer epoch retires. After a
    # round, this seat's main must BE that thread — same count,
    # same digest (#262's measure). Anything else means the
    # flagship thread disagrees with the set the round just
    # converged, and the verdict must say so rather than print
    # "converged" over it. Returns None when the arriving set
    # holds no main to judge against, else
    # {"ok": bool, "ours": n, "theirs": m}.
    files = (fetched or {}).get("files") or {}
    raw = files.get(SYNC_PREFIX + "chats.json")
    if raw is None:
        return None
    try:
        parsed = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return None
    remote_main = None
    for c in (parsed.get("chats") or []) if isinstance(parsed, dict) else []:
        rec = _chat_normalize(c)
        if rec is not None and _chat_is_main_family(rec):
            if str(rec["id"]) == MAIN_CHAT_ID or remote_main is None:
                remote_main = rec
    if remote_main is None:
        return None
    local_main = None
    for c in chats_all():
        if str(c.get("id")) == MAIN_CHAT_ID and not c.get("case"):
            local_main = c
            break
    epoch = max((local_main or {}).get("clearedAt") or 0,
                remote_main.get("clearedAt") or 0)
    expected, _added = _chat_merge_messages(
        (local_main or {}).get("messages"), remote_main.get("messages"))
    if epoch:
        expected = _chat_clear_filter(
            {"clearedAt": epoch, "messages": expected})
    ours = _sync_chat_substance(local_main or {})
    want = _sync_chat_substance({"messages": expected})
    return {"ok": ours == want, "ours": ours[0],
            "theirs": len(remote_main.get("messages") or [])}


def _sync_apply_pulse(fetched, summary):
    # v0.5.13e (#287): the pulse store converges like every
    # other citizen — watches by id (the newer updatedAt wins),
    # seen-sets and greeted sets unioned, hits merged by digest
    # (the newer read wins), batches by id. A watch deleted
    # here stays deleted only if the other seat is not holding
    # a NEWER copy — the same last-writer discipline the seat
    # merge keeps; there is no pulse tombstone in this build.
    raw = fetched["files"].get(SYNC_PREFIX + "pulse.json")
    if raw is None:
        return
    try:
        incoming = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return
    if not isinstance(incoming, dict):
        return
    inc = {"watches": [], "batches": [], "lastRun": 0, "greetedBatches": []}
    for w in (incoming.get("watches") or []):
        cw = _pulse_clean_watch(w)
        if cw is not None:
            inc["watches"].append(cw)
    for b in (incoming.get("batches") or []):
        cb = _pulse_clean_batch(b)
        if cb is not None:
            inc["batches"].append(cb)
    inc["greetedBatches"] = [str(x)[:160] for x in (incoming.get("greetedBatches") or [])]
    changed = 0
    with _pulse_lock:
        store = _pulse_load()
        by_id = {w["id"]: w for w in store["watches"]}
        for w in inc["watches"]:
            cur = by_id.get(w["id"])
            if cur is None:
                store["watches"].append(w)
                by_id[w["id"]] = w
                changed += 1
                continue
            base = w if (w.get("updatedAt") or 0) >= (cur.get("updatedAt") or 0) else cur
            merged = dict(base)
            seen = list(base.get("seen") or [])
            for d in (cur.get("seen") or []) + (w.get("seen") or []):
                if d not in seen:
                    seen.append(d)
            merged["seen"] = seen
            hits = {h["digest"]: h for h in (cur.get("hits") or [])}
            for h in (w.get("hits") or []):
                old = hits.get(h["digest"])
                if old is None or str(h.get("readAt") or "") >= str(old.get("readAt") or ""):
                    hits[h["digest"]] = h
            merged["hits"] = sorted(hits.values(),
                                    key=lambda h: str(h.get("readAt") or ""))[-PULSE_HITS_CAP:]
            merged["lastCheck"] = max(cur.get("lastCheck") or 0, w.get("lastCheck") or 0)
            if merged != cur:
                changed += 1
            store["watches"] = [merged if x["id"] == w["id"] else x
                                for x in store["watches"]]
            by_id[w["id"]] = merged
        have_batches = {b["id"] for b in store["batches"]}
        for b in inc["batches"]:
            if b["id"] not in have_batches:
                store["batches"].append(b)
                have_batches.add(b["id"])
                changed += 1
        store["batches"] = store["batches"][-PULSE_BATCHES_CAP:]
        greeted = list(store.get("greetedBatches") or [])
        for g in inc["greetedBatches"]:
            if g not in greeted:
                greeted.append(g)
        store["greetedBatches"] = greeted[-PULSE_BATCHES_CAP:]
        store["lastRun"] = max(store.get("lastRun") or 0,
                                incoming.get("lastRun") or 0)
        _pulse_save(store)
    summary["pulseChanged"] = summary.get("pulseChanged", 0) + changed


def _sync_apply_seat(fetched, summary):
    raw = fetched["files"].get(SYNC_PREFIX + "seat.json")
    if raw is None:
        return
    try:
        incoming = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return
    if not isinstance(incoming, dict):
        return
    _st, changes = _seat_merge(incoming, source="sync")
    summary["entriesPulled"] += changes["entriesAdded"]
    summary["entriesRemoved"] = summary.get("entriesRemoved", 0) + changes.get("entriesRemoved", 0)
    if changes["pictureChanged"] or changes["badgeChanged"]:
        summary["seatChanged"] = True


def sync_collect(cfg=None):
    # The local sync set as {path: bytes} plus the case index the
    # convergence reads the other side by.
    payload = chats_payload()
    # v0.4.10e (#252): the set carries a case and its chat
    # together — a chat whose case is not on this machine does
    # not ride at all, since it could only land on the far seat
    # as an orphan.
    export_chats = [c for c in payload["chats"]
                    if not c.get("case")
                    or _sync_case_files(str(c["case"])) is not None]
    files = {}
    files[SYNC_PREFIX + "chats.json"] = json.dumps(
        {"chats": export_chats, "uploads": payload["uploads"],
         "exportedAt": _sync_now_iso()}, indent=2).encode()
    state = _sync_state_load()
    files[SYNC_PREFIX + "tombstones.json"] = json.dumps(
        _sync_tombstones_pruned(state.get("tombstones") or {}), indent=2).encode()
    index = {}
    cases_root = os.path.join(VAULT_DIR, "cases")
    if os.path.isdir(cases_root):
        for slug in sorted(os.listdir(cases_root)):
            if not os.path.isdir(os.path.join(cases_root, slug)):
                continue
            entries = _sync_case_files(slug)
            if entries is None:
                continue
            # Build 2 (#199): the index badges a sensitive case
            # as sensitive — the receiving side (and any preview
            # of what a sync carries) can see the flag without
            # opening the manifest. The flag itself also rides
            # case.md, so it survives the round trip either way.
            _smeta = _case_manifest(slug) or {}
            index[slug] = {"files": entries,
                           "sensitive": str(_smeta.get("sensitive") or "").lower() in ("yes", "true", "1")}
            d = _case_dir(slug)
            for rel in entries:
                try:
                    with open(os.path.join(d, *rel.split("/")), "rb") as f:
                        files[SYNC_PREFIX + "cases/" + slug + "/" + rel] = f.read()
                except Exception:
                    continue
    files[SYNC_PREFIX + "cases-index.json"] = json.dumps(index, indent=2).encode()
    # The Sync Update's widened set (v0.3.0e Build 1, #200): the
    # ChipDeck as one unit, her two documents, and the seat state
    # — all anchored to the GitHub account, named in the
    # manifest so a set from a different account is refused at
    # the door, never half-applied.
    if cfg is None:
        cfg = load_config()
    deck_files, deck_index = _sync_deck_collect(cfg)
    files.update(deck_files)
    files[SYNC_PREFIX + "deck-index.json"] = json.dumps(deck_index, indent=2).encode()
    docs_index = {}
    for name, _label in _SYNC_DOCS:
        path = _sync_doc_path(name)
        try:
            with open(path, "rb") as f:
                data = f.read()
            files[SYNC_PREFIX + "docs/" + name + ".md"] = data
            docs_index[name] = {"sha": hashlib.sha256(data).hexdigest(),
                                "mtime": int(os.path.getmtime(path))}
        except Exception:
            continue
    files[SYNC_PREFIX + "docs-index.json"] = json.dumps(docs_index, indent=2).encode()
    files[SYNC_PREFIX + "seat.json"] = json.dumps(_seat_state_load(), indent=2).encode()
    # v0.5.13e (#287): the pulse store rides as account data.
    try:
        with open(PULSE_PATH, "rb") as f:
            files[SYNC_PREFIX + "pulse.json"] = f.read()
    except Exception:
        pass
    # v0.6.0e (#294): the day book rides as account data too —
    # one store, every seat, converged by identity with its
    # deletions tombstoned (see _sync_apply_daybook).
    try:
        with open(DAYBOOK_PATH, "rb") as f:
            files[SYNC_PREFIX + "daybook.json"] = f.read()
    except Exception:
        pass
    account = ""
    try:
        account = str((github_auth() or {}).get("login") or "")
    except Exception:
        account = ""
    # v0.5.13e (#262): the manifest declares SUBSTANCE, not
    # just counts — per-chat message counts and digests, and a
    # digest of the seat and pulse payloads — so the receiving
    # side verifies what arrived against what was sent, names
    # a shell a shell, and a later full arrival heals it.
    chat_digests = {}
    for c in export_chats:
        _n, _sha = _sync_chat_substance(c)
        chat_digests[str(c.get("id") or "")] = {"messages": _n, "sha": _sha}
    pulse_meta = {}
    if SYNC_PREFIX + "pulse.json" in files:
        try:
            _pw = json.loads(files[SYNC_PREFIX + "pulse.json"].decode("utf-8", "replace"))
            pulse_meta = {"watches": len((_pw or {}).get("watches") or []),
                          "sha": hashlib.sha256(files[SYNC_PREFIX + "pulse.json"]).hexdigest()}
        except Exception:
            pulse_meta = {}
    daybook_meta = {}
    if SYNC_PREFIX + "daybook.json" in files:
        try:
            _dw = json.loads(files[SYNC_PREFIX + "daybook.json"].decode("utf-8", "replace"))
            daybook_meta = {"entries": len((_dw or {}).get("entries") or []),
                            "sha": hashlib.sha256(files[SYNC_PREFIX + "daybook.json"]).hexdigest()}
        except Exception:
            daybook_meta = {}
    # v0.6.3e (#318): the main thread's clear marker rides the
    # manifest as well as the record, so a receiving seat can
    # read the epoch off the set's face before it converges.
    main_clear = {}
    for c in export_chats:
        if _chat_is_main_family(c) and c.get("clearedAt"):
            main_clear = {"clearedAt": c["clearedAt"],
                          "seat": c.get("clearedBy") or ""}
            break
    manifest = {"format": SYNC_FORMAT, "app": "Apex",
                "bridgeBuild": BRIDGE_BUILD, "syncedAt": _sync_now_iso(),
                "account": account,
                "mainClear": main_clear,
                "chats": len(export_chats), "cases": len(index),
                "deckFiles": len(deck_index), "docs": len(docs_index),
                "chatDigests": chat_digests,
                "chatsSha": hashlib.sha256(files[SYNC_PREFIX + "chats.json"]).hexdigest(),
                "seatSha": hashlib.sha256(files[SYNC_PREFIX + "seat.json"]).hexdigest(),
                "pulse": pulse_meta,
                "daybook": daybook_meta}
    files[SYNC_PREFIX + "manifest.json"] = json.dumps(manifest, indent=2).encode()
    return files, manifest, index


def _sync_repo(cfg):
    # (auth, base) for the operator's backup repo, ensuring it
    # exists — the same ensure github_run_backup performs. Returns
    # (None, fail) on a classified failure.
    auth = github_auth()
    if not auth:
        return None, None
    s = github_settings(cfg)
    login, token, repo = auth["login"], auth["token"], s["backup_repo"]
    base = "https://api.github.com/repos/" + login + "/" + repo
    st, _ = github_api("GET", base, token)
    if st == 404:
        st, created = github_api("POST", "https://api.github.com/user/repos", token, {
            "name": repo, "private": s["backup_private"],
            "description": "Apex backup — her ready-to-fly essentials (Soul, Memory, settings, chips, RAG packs, workflows, the Vault, Mesh pairing cards, chats).",
            "auto_init": True})
        if st not in (200, 201):
            return None, _sync_fail(st, "create the backup repo")
    elif st != 200:
        return None, _sync_fail(st, "read the backup repo")
    return (auth, base), None


def sync_fetch(cfg):
    # The remote sync set, through git trees/blobs like the backup
    # import (so large evidence files still arrive whole). A repo
    # with no sync/ tree yet is an EMPTY remote, not an error.
    # Returns ({"files": {...}, "manifest": {...}|None}, None) or
    # (None, fail).
    got = _sync_repo(cfg)
    if got[0] is None:
        return None, got[1] or _sync_auth_error()
    auth, base = got[0]
    token = auth["token"]
    st, repo_j = github_api("GET", base, token)
    if st != 200 or not isinstance(repo_j, dict):
        return None, _sync_fail(st, "read the backup repo")
    branch = str(repo_j.get("default_branch") or "main")
    st, tree = github_api("GET", base + "/git/trees/" + urllib.parse.quote(branch, safe="") + "?recursive=1", token)
    if st != 200 or not isinstance(tree, dict):
        return None, _sync_fail(st, "list the backup repo's files")
    blobs = {}
    for entry in tree.get("tree") or []:
        if entry.get("type") == "blob" and str(entry.get("path") or "").startswith(SYNC_PREFIX):
            blobs[str(entry["path"])] = entry.get("sha")
    files = {}
    for path, sha in blobs.items():
        bst, blob = github_api("GET", base + "/git/blobs/" + str(sha), token)
        if bst == 200 and isinstance(blob, dict) and blob.get("content"):
            try:
                files[path] = base64.b64decode(str(blob["content"]))
            except Exception:
                continue
    manifest = None
    if SYNC_PREFIX + "manifest.json" in files:
        try:
            parsed = json.loads(files[SYNC_PREFIX + "manifest.json"].decode("utf-8", "replace"))
            if isinstance(parsed, dict) and parsed.get("format") in (SYNC_FORMAT, "navi-sync/1"):
                manifest = parsed
        except Exception:
            manifest = None
    if manifest is not None:
        acct = str(manifest.get("account") or "")
        mine = str((github_auth() or {}).get("login") or "")
        if not acct or not mine:
            # v0.6.4e (#323): defense in depth on the repo road —
            # a set the door cannot attribute to an account is
            # not passed through on the strength of repo access
            # alone. (A repo with no manifest at all is still an
            # empty remote, above; this is a set that NAMES
            # itself sync and names no owner.)
            return None, {"ok": False, "kind": "auth", "state": "auth",
                          "route": "Backup repo",
                          "error": "That sync set names no GitHub account, so it cannot be attributed — sync is anchored to the account, and a set that cannot be attributed does not cross. Nothing was applied."}
        if acct.lower() != mine.lower():
            return None, {"ok": False, "kind": "auth", "state": "auth",
                          "route": "Backup repo",
                          "error": "That sync set belongs to a different GitHub account (" + acct + ") — sync is anchored to the account, and this machine is signed in as " + mine + ". Nothing crossed. Sign in with the account that owns the set, or let this machine start a set of its own."}
    return {"files": files, "manifest": manifest}, None


def _sync_reconcile_case_chats(remote_case_slugs, summary):
    # v0.4.10e (#252): a case and its chat travel together, so
    # a seat never HOLDS a case chat whose case is not on its
    # desk — the field proof was a "Case — Test" chat listed
    # while the desk read empty. Once the arriving cases have
    # landed, any chat whose case is absent folds away — not
    # listed, not offered — with a receipt in the sync state
    # naming the gap: the case, the chat, and whether the case
    # was in the arriving set and failed to land or never
    # crossed at all. No tombstone is recorded: the fold is
    # not the operator's delete, and if a later set brings the
    # case whole, the chat rides with it and the pair stands
    # again. An identical repeat arrival folds again but does
    # not receipt twice.
    remote_case_slugs = remote_case_slugs or set()
    state = _sync_state_load()
    receipts = state.get("orphanCaseChats") or []
    known = {(str(r.get("chatId")), r.get("updatedAt")) for r in receipts}
    gap_slugs = {str(r.get("case")) for r in receipts}
    dirty = False
    for chat in chats_all():
        slug = chat.get("case")
        if not slug:
            continue
        if _sync_case_files(str(slug)) is not None:
            summary["casePairsWhole"] += 1
            if str(slug) in gap_slugs:
                # v0.4.10e (#254's receipt): a gap an earlier
                # round receipted is healed — the pair stands
                # whole again. A repair is a delta, counted
                # once; the old receipt is cleared.
                summary["casePairsRepaired"] += 1
                receipts = [r for r in receipts if str(r.get("case")) != str(slug)]
                gap_slugs.discard(str(slug))
                dirty = True
            continue
        key = (str(chat["id"]), chat.get("updatedAt"))
        chat_delete(str(chat["id"]), _record=False)
        summary["orphanedCaseChats"] += 1
        summary["caseChatGapTitles"].append(str(chat.get("title") or slug))
        if key not in known:
            receipts.append({
                "chatId": str(chat["id"]), "case": str(slug),
                "title": str(chat.get("title") or ""),
                "messages": len(chat.get("messages") or []),
                "updatedAt": chat.get("updatedAt"),
                "at": time.time(),
                "reason": ("case-in-set-failed-to-land"
                           if str(slug) in remote_case_slugs
                           else "case-not-in-set")})
            known.add(key)
            dirty = True
        summary["eyes"].append("the chat of a case that did not cross: “" +
                               str(chat.get("title") or slug) + "”")
    if dirty:
        state["orphanCaseChats"] = receipts[-20:]
        _sync_state_save(state)
    return summary


def _sync_apply_tombstones(tombs, summary):
    # Deletions land BEFORE anything converges in — and only over
    # copies no newer than the deletion itself. Returns the set of
    # tombstone keys that were SUPERSEDED by newer local work (the
    # caller drops them from the stored set).
    dropped = {"chats": set(), "cases": set(), "vault": set(),
               "dossier": set()}
    chat_tombs = tombs.get("chats") or {}
    if MAIN_CHAT_ID in {str(k) for k in chat_tombs}:
        # v0.3.0e Build 1 (#200): the main chat cannot be
        # deleted, so its tombstone can never land — drop it
        # from the set rather than retry it every round.
        dropped["chats"].add(MAIN_CHAT_ID)
    for rec in chats_all():
        cid = str(rec["id"])
        if cid == MAIN_CHAT_ID:
            continue
        # v0.4.10e (#253): a chat tombstone is a fact, not a
        # suggestion weighed against timestamps. The named
        # thread dies, and so does any local representative of
        # its conversation (the "conv:" keys chat_delete
        # records), however new its last line is — newer work
        # inside a deleted conversation is not a resurrection
        # door. The operator deleted the conversation, not a
        # snapshot of it. (Build 5 weighed updatedAt here and
        # let the newer copy keep the thread — and the delete
        # the operator watched happen quietly undone itself.)
        if cid in chat_tombs or _chat_conv_tomb_key(rec) in chat_tombs:
            if chat_delete(cid, _record=False):
                summary["removedChats"] += 1
    for slug, at in (tombs.get("cases") or {}).items():
        entries = _sync_case_files(slug)
        if entries is None:
            continue
        newest = max([e["mtime"] for e in entries.values()] or [0])
        if newest <= at:
            res, _err = case_delete(slug)
            if res is not None:
                summary["removedCases"] += 1
        else:
            dropped["cases"].add(str(slug))
    for rel, at in (tombs.get("vault") or {}).items():
        safe = _sync_safe_rel(rel)
        if safe is None:
            continue
        # A stale local copy goes (case files included — and the
        # converge below vetoes re-writing this same file from
        # the remote copy it was deleted against); a file edited
        # after the deletion is newer work and keeps its copy,
        # dropping the tombstone.
        full = vault_fs_path(safe)
        try:
            if os.path.isfile(full):
                if os.path.getmtime(full) <= at:
                    vault_delete_note(safe)
                    summary["removedNotes"] += 1
                else:
                    dropped["vault"].add(safe)
        except Exception:
            continue
    # v0.5.9e (#280): dossier element tombstones — an element
    # deleted on the other seat dies here too, everywhere it
    # stands (the store, this seat's case findings, the
    # formations), with custody lines written by the same
    # funnel a local delete uses. An element tombstone is a
    # fact, like a chat's (#253): it is never superseded by
    # newer work, because a re-run mints a NEW element — the
    # deleted one's identity never legitimately returns. The
    # finding files the purge removes are vault-tombstoned in
    # the same act, and those tombstones join this round's set
    # so the converge below cannot re-write the files from the
    # remote copy in the very round that deleted them.
    dossier_tombs = tombs.get("dossier") or {}
    if dossier_tombs:
        try:
            with DOSSIER_LOCK:
                _dz_store = _dossier_store()
                _dz_held = {e["id"] for d in _dz_store["dossiers"]
                            for e in d.get("elements") or []}
                for eid in dossier_tombs:
                    if eid not in _dz_held:
                        continue
                    _res, _err = element_delete(
                        eid, "sync", "", record_tombstone=False)
                    if _res:
                        summary["removedElements"] = \
                            summary.get("removedElements", 0) + 1
                        for _rel in _res.get("vaultTombstones") or []:
                            tombs.setdefault("vault", {})[_rel] = \
                                time.time()
        except Exception:
            pass
    return dropped


def _sync_apply_cases(fetched, tombs, summary):
    files = fetched["files"]
    remote_index = {}
    if SYNC_PREFIX + "cases-index.json" in files:
        try:
            parsed = json.loads(files[SYNC_PREFIX + "cases-index.json"].decode("utf-8", "replace"))
            if isinstance(parsed, dict):
                remote_index = parsed
        except Exception:
            remote_index = {}
    day = time.strftime("%Y-%m-%d", time.localtime())
    vault_tombs = tombs.get("vault") or {}
    for slug in sorted(remote_index):
        if not re.match(r"^[a-z0-9][a-z0-9-]*$", str(slug)):
            continue
        if str(slug) in (tombs.get("cases") or {}):
            continue  # tombstoned in the phase above; nothing lands
        remote_files = (remote_index.get(slug) or {}).get("files") or {}
        local_before = _sync_case_files(slug)
        d = _case_dir(slug)
        if d is None:
            continue
        wrote = 0
        for rel in sorted(remote_files):
            safe = _sync_safe_rel(rel)
            if safe is None:
                continue
            meta = remote_files.get(rel) or {}
            rsha = str(meta.get("sha") or "")
            rmtime = float(meta.get("mtime") or 0)
            veto = vault_tombs.get("cases/" + slug + "/" + safe)
            if veto is not None and veto >= rmtime:
                continue  # deleted here after that copy was made
            data = files.get(SYNC_PREFIX + "cases/" + slug + "/" + safe)
            if data is None:
                continue
            full = os.path.join(d, *safe.split("/"))
            local_meta = (local_before or {}).get(safe)
            if local_meta and local_meta["sha"] == rsha:
                continue
            if local_meta and local_meta["mtime"] > rmtime + 1:
                continue  # this machine's copy is the newer work
            if local_meta and rmtime <= local_meta["mtime"] + 1:
                # Same-second divergence: park the remote copy as
                # a dated sibling — both survive, the case is
                # named to the operator's eyes.
                sib = full + ".from-sync-" + day
                try:
                    with open(sib, "rb") as f:
                        if f.read() == data:
                            continue
                except Exception:
                    pass
                os.makedirs(os.path.dirname(full), exist_ok=True)
                with open(sib, "wb") as f:
                    f.write(data)
                if slug not in summary["caseEyes"]:
                    summary["caseEyes"].append(slug)
                continue
            os.makedirs(os.path.dirname(full), exist_ok=True)
            with open(full, "wb") as f:
                f.write(data)
            try:
                os.utime(full, (rmtime, rmtime))
            except Exception:
                pass
            wrote += 1
        if wrote:
            if local_before is None:
                summary["casesAdded"] += 1
            else:
                summary["casesUpdated"] += 1
            summary["caseFilesPulled"] += wrote


def sync_apply(cfg, fetched):
    # Converge a fetched remote sync set into this machine:
    # tombstones, then chats, then case files. Returns the round
    # summary the card speaks.
    summary = {"removedChats": 0, "removedCases": 0, "removedNotes": 0,
               "chatsAdded": 0, "chatsReplaced": 0, "chatsSiblings": 0,
               "chatsDeduped": 0, "chatsMainMerged": 0, "chatsMainFolded": 0,
               "chatsMainMergedMessages": 0, "chatsMainFoldedMessages": 0,
               "orphanedCaseChats": 0, "casePairsWhole": 0,
               "casePairsRepaired": 0, "caseChatGapTitles": [],
               "casesAdded": 0, "casesUpdated": 0, "caseFilesPulled": 0,
               "deckFilesPulled": 0, "cardsImported": 0, "docsPulled": 0,
               "entriesPulled": 0, "seatChanged": False,
               "pulseChanged": 0, "daybookChanged": 0, "incomplete": [],
               "eyes": [], "caseEyes": []}
    files = fetched["files"]
    remote_tombs = {}
    if SYNC_PREFIX + "tombstones.json" in files:
        try:
            parsed = json.loads(files[SYNC_PREFIX + "tombstones.json"].decode("utf-8", "replace"))
            if isinstance(parsed, dict):
                remote_tombs = parsed
        except Exception:
            remote_tombs = {}
    state = _sync_state_load()
    merged = _sync_merge_tombstones(state.get("tombstones") or {}, remote_tombs)
    dropped = _sync_apply_tombstones(merged, summary)
    remote_chats, remote_uploads = [], None
    if SYNC_PREFIX + "chats.json" in files:
        try:
            parsed = json.loads(files[SYNC_PREFIX + "chats.json"].decode("utf-8", "replace"))
            if isinstance(parsed, dict):
                if isinstance(parsed.get("chats"), list):
                    remote_chats = parsed["chats"]
                if isinstance(parsed.get("uploads"), list):
                    remote_uploads = parsed["uploads"]
        except Exception:
            remote_chats, remote_uploads = [], None
    # v0.5.13e (#262): the substance law. The sender's manifest
    # declares what the set SHOULD hold; arrivals are verified
    # against it before anything converges. A payload whose
    # bytes do not match its declared digest arrived as a
    # shell: it is named incomplete, and for the seat and the
    # pulse — single documents, merged destructively — the
    # shell is refused outright rather than merged over good
    # local substance. Sets from older builds declare nothing
    # and are taken as they come, as before.
    remote_manifest = {}
    if SYNC_PREFIX + "manifest.json" in files:
        try:
            _pm = json.loads(files[SYNC_PREFIX + "manifest.json"].decode("utf-8", "replace"))
            if isinstance(_pm, dict):
                remote_manifest = _pm
        except Exception:
            remote_manifest = {}
    declared_chats = remote_manifest.get("chatDigests")
    if not isinstance(declared_chats, dict):
        declared_chats = {}
    _seat_shell = False
    if remote_manifest.get("seatSha") and SYNC_PREFIX + "seat.json" in files:
        if hashlib.sha256(files[SYNC_PREFIX + "seat.json"]).hexdigest() != remote_manifest["seatSha"]:
            _seat_shell = True
            summary["incomplete"].append(
                {"kind": "seat", "id": "seat",
                 "name": "Settings & theme (the seat)"})
    _pulse_shell = False
    _pulse_decl = remote_manifest.get("pulse") if isinstance(remote_manifest.get("pulse"), dict) else {}
    if _pulse_decl.get("sha") and SYNC_PREFIX + "pulse.json" in files:
        if hashlib.sha256(files[SYNC_PREFIX + "pulse.json"]).hexdigest() != _pulse_decl["sha"]:
            _pulse_shell = True
            summary["incomplete"].append(
                {"kind": "pulse", "id": "pulse", "name": "Pulse watches"})
    _daybook_shell = False
    _daybook_decl = remote_manifest.get("daybook") if isinstance(remote_manifest.get("daybook"), dict) else {}
    if _daybook_decl.get("sha") and SYNC_PREFIX + "daybook.json" in files:
        if hashlib.sha256(files[SYNC_PREFIX + "daybook.json"]).hexdigest() != _daybook_decl["sha"]:
            _daybook_shell = True
            summary["incomplete"].append(
                {"kind": "daybook", "id": "daybook", "name": "Her working day"})
    # The tombstone's other half: a chat ABSENT here but present
    # in the remote set is not an "add" when a tombstone covers
    # it — it is the resurrection the tombstone exists to refuse.
    # (The phase above only deletes copies that exist locally;
    # without this filter, convergence would faithfully re-add
    # every deleted thread from the other machine's store.)
    # A tombstone covers its CONVERSATION as well as its id
    # (the "conv:" keys chat_delete records). Through
    # v0.4.9e a copy stamped newer than the tombstone was let
    # back in as a revival generation; v0.4.10e (#253) retires
    # that door — covered is covered, at any stamp.
    chat_tombs = merged.get("chats") or {}
    if chat_tombs:
        kept = []
        for raw in remote_chats:
            rec = _chat_normalize(raw)
            if rec is None:
                continue
            if _chat_is_main_family(rec):
                # v0.4.10e (#251): the main seat is governed by
                # the merge, never by tombstones — it cannot be
                # deleted, and its records fold into the one
                # thread at convergence below.
                kept.append(raw)
                continue
            if str(rec["id"]) in chat_tombs or \
                    _chat_conv_tomb_key(rec) in chat_tombs:
                # v0.4.10e (#253): covered by a tombstone —
                # refused, however new its last line claims to
                # be. Build 5's revival generation is retired:
                # a deleted conversation does not come back
                # wearing a "(from sync)" coat, and a seat that
                # was away when the delete happened cannot undo
                # it by syncing late. The tombstones themselves
                # are kept: they are the record of the
                # operator's act, pruned only by the standing
                # 90-day rule in _sync_tombstones_pruned.
                continue
            kept.append(raw)
        remote_chats = kept
    for kind in ("chats", "cases", "vault", "dossier"):
        for key in dropped[kind]:
            merged[kind].pop(key, None)
    state["tombstones"] = merged
    _sync_state_save(state)
    # v0.5.13e (#262): per-chat verification. A remote chat
    # whose substance (count + digest) does not match what the
    # manifest declared for it arrived as a shell. Where this
    # seat already holds MORE of that conversation, the shell
    # is refused — convergence never trades substance for a
    # husk; where it holds nothing, the shell lands (a husk is
    # better than a hole) and is still named, so the bar and
    # the record say plainly that it is incomplete until a
    # fuller arrival heals it.
    if declared_chats:
        _local_counts = {}
        for _c in chats_all():
            _local_counts[str(_c.get("id") or "")] = len(_c.get("messages") or [])
        _kept = []
        for _raw in remote_chats:
            _rec = _chat_normalize(_raw)
            if _rec is None:
                _kept.append(_raw)
                continue
            _decl = declared_chats.get(str(_rec["id"]))
            if not isinstance(_decl, dict):
                _kept.append(_raw)
                continue
            _n, _sha = _sync_chat_substance(_rec)
            if _n == _decl.get("messages") and _sha == _decl.get("sha"):
                _kept.append(_raw)
                continue
            summary["incomplete"].append(
                {"kind": "chat", "id": str(_rec["id"]),
                 "name": "Chat \u201c" + str(_rec.get("title") or _rec["id"]) + "\u201d"})
            if _local_counts.get(str(_rec["id"]), 0) > _n:
                continue
            _kept.append(_raw)
        remote_chats = _kept
    before_ids = {str(c["id"]) for c in chats_all()}
    conv = chats_converge(remote_chats, remote_uploads)
    summary["chatsAdded"] = conv["added"]
    summary["chatsReplaced"] = conv["replaced"]
    summary["chatsSiblings"] = conv["siblings"]
    summary["chatsDeduped"] = conv.get("deduped", 0)
    summary["chatsMainMerged"] = conv.get("mainMerged", 0)
    summary["chatsMainFolded"] = conv.get("mainFolded", 0)
    summary["chatsMainMergedMessages"] = conv.get("mainMergedMessages", 0)
    summary["chatsMainFoldedMessages"] = conv.get("mainFoldedMessages", 0)
    if conv["siblings"]:
        for c in chats_all():
            if str(c["id"]) not in before_ids and "-sync-" in str(c["id"]):
                summary["eyes"].append(str(c.get("title") or c["id"]))
    # v0.5.13e (#262): the arrival proof, read back off the
    # local store — after convergence, every declared chat
    # must stand here with at least the messages it was sent
    # with. Convergence unions and never deletes, so a short
    # count means the substance genuinely did not cross.
    if declared_chats:
        _after = {str(c.get("id") or ""): len(c.get("messages") or [])
                  for c in chats_all()}
        _named = {e["id"] for e in summary["incomplete"] if e["kind"] == "chat"}
        for _cid, _decl in declared_chats.items():
            if not isinstance(_decl, dict) or _cid in _named:
                continue
            if _after.get(_cid, 0) < int(_decl.get("messages") or 0):
                _ttl = next((str(c.get("title") or _cid) for c in chats_all()
                             if str(c.get("id") or "") == _cid), _cid)
                summary["incomplete"].append(
                    {"kind": "chat", "id": _cid,
                     "name": "Chat \u201c" + _ttl + "\u201d"})
    _sync_apply_cases(fetched, merged, summary)
    # v0.4.10e (#252): the pairing check runs once the arriving
    # cases have landed — a chat whose case is not on this desk
    # folds away with a receipt and the gap is named, rather
    # than listing as a case chat whose case is missing.
    _remote_case_slugs = set()
    if SYNC_PREFIX + "cases-index.json" in files:
        try:
            _parsed_index = json.loads(files[SYNC_PREFIX + "cases-index.json"].decode("utf-8", "replace"))
            if isinstance(_parsed_index, dict):
                _remote_case_slugs = {str(k) for k in _parsed_index}
        except Exception:
            pass
    _sync_reconcile_case_chats(_remote_case_slugs, summary)
    _sync_apply_deck(cfg, fetched, summary)
    _sync_apply_docs(fetched, summary, state)
    _sync_state_save(state)
    if not _pulse_shell:
        _sync_apply_pulse(fetched, summary)
    if not _daybook_shell:
        _sync_apply_daybook(fetched, summary, merged)
    if not _seat_shell:
        _sync_apply_seat(fetched, summary)
    # v0.6.4e (#323): attribution — the apply path records, per
    # item it converged, the account the arriving set named as
    # its own ("" when it named none). The ledger is the purge's
    # evidence (see sync_account_gate_sweep): from this build
    # on, every converged item can be attributed to the set
    # that carried it, exactly, at the moment it landed.
    try:
        _src_acct = str(remote_manifest.get("account") or "")
        if not _src_acct and isinstance(fetched.get("manifest"), dict):
            _src_acct = str(fetched["manifest"].get("account") or "")
        _st3 = _sync_state_load()
        _attr = _st3.get("attribution") if isinstance(_st3.get("attribution"), dict) else {}
        for _fam in ("chats", "cases", "docs", "daybook"):
            if not isinstance(_attr.get(_fam), dict):
                _attr[_fam] = {}
        for _c in remote_chats:
            _rec = _chat_normalize(_c)
            if _rec is not None:
                _attr["chats"][str(_rec["id"])] = _src_acct
        for _slug in _remote_case_slugs:
            _attr["cases"][str(_slug)] = _src_acct
        for _name, _label in _SYNC_DOCS:
            if SYNC_PREFIX + "docs/" + _name + ".md" in files:
                _attr["docs"][_name] = _src_acct
        _db_raw = files.get(SYNC_PREFIX + "daybook.json")
        if _db_raw:
            try:
                _db = json.loads(_db_raw.decode("utf-8", "replace"))
                for _e in ((_db or {}).get("entries") or []):
                    if isinstance(_e, dict) and _e.get("id"):
                        _attr["daybook"][str(_e["id"])] = _src_acct
            except Exception:
                pass
        _st3["attribution"] = _attr
        _sync_state_save(_st3)
    except Exception:
        pass
    summary["changed"] = (summary["removedChats"] + summary["removedCases"]
                          + summary["removedNotes"] + summary["chatsAdded"]
                          + summary["chatsReplaced"] + summary["chatsSiblings"]
                          + summary["chatsDeduped"] + summary["chatsMainMerged"]
                          + summary["chatsMainFolded"] + summary["orphanedCaseChats"]
                          + summary["casesAdded"] + summary["casesUpdated"]
                          + summary["deckFilesPulled"] + summary["cardsImported"]
                          + summary["docsPulled"] + summary["entriesPulled"]
                          + summary.get("entriesRemoved", 0)
                          + (1 if summary["seatChanged"] else 0)
                          + summary.get("pulseChanged", 0)
                          + summary.get("daybookChanged", 0))
    # v0.5.13e (#262): the incomplete record persists in the
    # sync state — the bar names shells until a later round's
    # full arrival heals them, which is simply a round where
    # they are not named again.
    try:
        _st2 = _sync_state_load()
        _prev = [e for e in (_st2.get("incomplete") or [])
                 if isinstance(e, dict) and e.get("id")]
        _now_ids = {(e["kind"], e["id"]) for e in summary["incomplete"]}
        _still = [e for e in _prev
                  if (e.get("kind"), e.get("id")) in _now_ids]
        _have = {(e.get("kind"), e.get("id")) for e in _still}
        for e in summary["incomplete"]:
            if (e["kind"], e["id"]) not in _have:
                _still.append(e)
                _have.add((e["kind"], e["id"]))
        _st2["incomplete"] = _still
        _sync_state_save(_st2)
    except Exception:
        pass
    # v0.6.3e (#318): the main thread's verdict, judged AFTER
    # everything landed — the finish summary renames the round
    # on a mismatch, so no road (repo, Mesh, pull, receive) can
    # print a clean "converged" over a divergent main thread.
    try:
        summary["mainVerdict"] = _sync_main_verdict(fetched)
    except Exception:
        summary["mainVerdict"] = None
    return summary


def _sync_push(cfg, base, token, files, fetched_files):
    # Publish the converged local set. Files the remote already
    # holds byte-identical are not rewritten. Returns the count
    # of files actually written, or (None, fail).
    # v0.4.10e (#254): the write phase reports its counts to
    # the sync bar — the differing-file total is known before
    # the loop, and the written count as it goes (throttled:
    # at most twice a second, plus the final state).
    total = sum(1 for p in files if fetched_files.get(p) != files[p])
    _sync_progress_set("write", counts={"filesTotal": total, "filesWritten": 0})
    written = 0
    last_note = 0.0
    for path in sorted(files):
        if fetched_files.get(path) == files[path]:
            continue
        url = base + "/contents/" + urllib.parse.quote(path, safe="/")
        sha = None
        gst, existing = github_api("GET", url, token)
        if gst == 200 and isinstance(existing, dict):
            sha = existing.get("sha")
        elif gst not in (200, 404):
            return None, _sync_fail(gst, "check " + path + " in the backup repo")
        body = {"message": "Apex Quick Sync " + _sync_now_iso(),
                "content": base64.b64encode(files[path]).decode()}
        if sha:
            body["sha"] = sha
        pst, _resp = github_api("PUT", url, token, body)
        if pst not in (200, 201):
            return None, _sync_fail(pst, "write " + path + " to the backup repo")
        written += 1
        if time.time() - last_note >= 0.5 or written == total:
            last_note = time.time()
            _sync_progress_set("write", counts={"filesTotal": total, "filesWritten": written})
    return written, None


def _sync_record_result(state, trigger, result):
    state["lastSyncAt"] = _sync_now_iso()
    state["lastSyncEpoch"] = time.time()
    state["lastTrigger"] = trigger
    state["lastState"] = result.get("state") or ""
    state["lastResult"] = result.get("message") or result.get("error") or ""
    # v0.4.10e (#254): the durable receipt — the sync bar's
    # landing face reads this record, and the #242 producer
    # announces from it. Counts ride the finish summary's
    # receipt; failure verdicts carry their message and no
    # counts. The record is the copy that outlives the bar.
    receipt = result.get("receipt") if isinstance(result.get("receipt"), dict) else {}
    state["lastReceipt"] = {
        "at": state["lastSyncEpoch"],
        "trigger": trigger,
        "route": result.get("route") or "",
        "ok": result.get("ok") is True,
        "state": result.get("state") or "",
        "message": result.get("message") or result.get("error") or "",
        "counts": receipt,
        # v0.6.2e (#303): a directional round (Send / Receive)
        # names its direction on the durable receipt too.
        "direction": result.get("direction") or "",
        # v0.6.3e (#317): a round the operator's choice stopped
        # names the supersession on its record.
        "preempted": bool(result.get("preempted")),
        # v0.6.3e (#318): a renamed verdict carries the main
        # thread's disagreement on the receipt too.
        "mainThread": result.get("mainThread") or None,
        # v0.6.4e (#323): a Mesh-road account refusal the round
        # met is named on the receipt — the fall-through to the
        # repo road is never a silent success shape.
        "meshRefusal": result.get("meshRefusal") or _sync_mesh_refusal_take(),
        "material": bool(receipt.get("material")),
        "roundId": result.get("roundId") or "",
        # v0.5.1e (#257): the round's measured duration rides
        # the receipt — the bar names the road and the time it
        # actually took, measured, never asserted.
        "durationMs": (result.get("durationMs")
                       if isinstance(result.get("durationMs"), (int, float)) else None),
    }
    _sync_state_save(state)


def _sync_finish_summary(summary, pushed_files, open_chat_id, collected_chats):
    eyes = list(summary["eyes"])
    for slug in summary["caseEyes"]:
        meta = _case_manifest(slug)
        eyes.append("the case “" + str((meta or {}).get("title") or slug) + "”")
    pulled_chats = summary["chatsAdded"] + summary["chatsReplaced"]
    result = {
        "ok": True,
        "state": "eyes" if eyes else "synced",
        "route": "Backup repo",
        "pulledChats": pulled_chats,
        "pushedFiles": pushed_files,
        "caseFilesPulled": summary["caseFilesPulled"],
        "deckFilesPulled": summary.get("deckFilesPulled", 0),
        "cardsImported": summary.get("cardsImported", 0),
        "ghostRefusals": list(summary.get("ghostRefusals") or []),
        "docsPulled": summary.get("docsPulled", 0),
        "entriesPulled": summary.get("entriesPulled", 0),
        "entriesRemoved": summary.get("entriesRemoved", 0),
        "seatChanged": bool(summary.get("seatChanged")),
        "incomplete": [str(e.get("name") or e.get("id") or "")
                       for e in (summary.get("incomplete") or [])],
        "removed": summary["removedChats"] + summary["removedCases"] + summary["removedNotes"],
        "needsEyes": eyes,
        "openIncluded": bool(open_chat_id) and str(open_chat_id) in collected_chats,
    }
    if eyes:
        result["message"] = ("Synced — but " + str(len(eyes)) + " item"
                             + ("s" if len(eyes) != 1 else "")
                             + " parked both copies and need your eyes: "
                             + "; ".join(eyes[:3]) + ".")
    elif result["incomplete"]:
        result["message"] = ("Synced — but " + str(len(result["incomplete"]))
                             + " item" + ("s" if len(result["incomplete"]) != 1 else "")
                             + " arrived incomplete: "
                             + "; ".join(result["incomplete"][:3])
                             + ". A fuller copy heals "
                             + ("them" if len(result["incomplete"]) != 1 else "it")
                             + " on a later round.")
    elif pulled_chats or summary["chatsSiblings"] or summary["changed"]:
        result["message"] = "Synced — the machines converged."
    else:
        result["message"] = "Synced — already in step."
    # v0.6.3e (#318): the verdict honesty gate. When the main
    # thread disagrees with the converged set after the round,
    # the verdict is RENAMED — never a clean "converged", never
    # a plain "Synced" — on this result, and therefore on the
    # durable receipt and the Sync card's last-round line that
    # read it back. The direction is named in counts: theirs is
    # the arriving set's main, ours is this seat's.
    _mv = summary.get("mainVerdict")
    if isinstance(_mv, dict) and not _mv.get("ok"):
        result["mainThread"] = {"ours": _mv.get("ours"),
                                 "theirs": _mv.get("theirs")}
        result["state"] = "partial"
        result["message"] = (
            "Mostly synced — the main thread disagrees (theirs "
            + str(_mv.get("theirs")) + " messages, ours "
            + str(_mv.get("ours")) + "). The rest of the set "
            "converged; the main thread did not, and this round "
            "will not call it converged.")
    # v0.4.10e (#254): the round's receipt, in the operator's
    # terms — what the sync bar hangs off the bar on landing
    # and what the #242 announcement is composed from. A round
    # is MATERIAL when the sync family actually moved the
    # room's shape: ghost threads folded (#251), a case/chat
    # pairing repaired or a gap folded away (#252), or another
    # seat's tombstone honored here (#253). Ordinary arrivals
    # and departures are traffic, not material.
    receipt = {
        "mergedMessages": summary.get("chatsMainMergedMessages", 0),
        "foldedGhosts": summary.get("chatsMainFolded", 0),
        "foldedMessages": summary.get("chatsMainFoldedMessages", 0),
        "pairsWhole": summary.get("casePairsWhole", 0),
        "pairsRepaired": summary.get("casePairsRepaired", 0),
        "gapTitles": list(summary.get("caseChatGapTitles") or []),
        "tombstones": summary.get("removedChats", 0),
        "removedCases": summary.get("removedCases", 0),
        "removedNotes": summary.get("removedNotes", 0),
        "chatsAdded": summary.get("chatsAdded", 0),
        "chatsReplaced": summary.get("chatsReplaced", 0),
        "changed": summary.get("changed", 0),
    }
    receipt["incomplete"] = list(result["incomplete"])
    receipt["material"] = bool(receipt["foldedGhosts"] or receipt["gapTitles"]
                               or receipt["pairsRepaired"] or receipt["tombstones"]
                               or receipt["incomplete"])
    result["receipt"] = receipt
    return result


# -- Quick Sync's Mesh transport, borrower side (Build 6, #176) ------
# A round tries the Mesh first: for each paired peer this machine
# borrows from, in card order, a probe asks whether the road is
# lit; the first answering peer gets the exchange — one whole
# signed request carrying this machine's set out and the peer's
# set back, converged on arrival. Any failure at any step falls
# through to the backup-repo round, unchanged. The set is capped:
# past the cap the repo (built for bulk) carries the round.
_SYNC_MESH_MAX_BYTES = 96 * 1024 * 1024


def _sync_mesh_peers():
    ms = mesh_settings(load_config())
    if not ms["enabled"]:
        return []
    return [p for p in ms["borrow"]["peers"] if p["canBorrow"]]


# -- The account gate, initiator side (v0.6.4e, #323) -------------
# A set is offered to a peer only after the handshake proves
# the peer is a SEAT OF THIS ACCOUNT: the peer states its
# GitHub login at /mesh/sync/hello, and the logins must be
# present and equal. Pairing proves the road, never the
# account — peers are other operators by design; seats are
# this operator's own machines. A refusal is remembered for
# the round's receipt (the fall-through to the repo road is
# correct behavior for the operator's own seats, but it is
# never silent about why the Mesh road was not used).
_SYNC_MESH_REFUSAL = {"text": ""}


def _sync_mesh_refusal_note(text):
    if text and not _SYNC_MESH_REFUSAL["text"]:
        _SYNC_MESH_REFUSAL["text"] = text


def _sync_mesh_refusal_take():
    text = _SYNC_MESH_REFUSAL["text"]
    _SYNC_MESH_REFUSAL["text"] = ""
    return text


def _sync_mesh_hello(peer):
    # The peer's stated account, or None when the handshake
    # itself did not answer (a dark road, or a build too old
    # to state one — either way, nothing it says can prove it
    # a seat, and no set is offered to it).
    payload = _sync_mesh_ask(peer, "/mesh/sync/hello", {})
    if payload is None:
        return None
    return str(payload.get("account") or "")


def _sync_mesh_peer_gate(peer):
    # "" when the peer is proven a seat of this account; else
    # the refusal sentence the receipt carries. Called only
    # for peers whose road probed lit.
    mine = str((github_auth() or {}).get("login") or "")
    if not mine:
        return ("Mesh road refused: this seat's own account is "
                "unknown — sign in so Sync can prove which "
                "machines are its seats.")
    theirs = _sync_mesh_hello(peer)
    if theirs is None or not theirs:
        return ("Mesh road refused: " + str(peer.get("name") or "the peer")
                + " would not state an account — Sync cannot "
                "prove it is a seat of this account.")
    if theirs.lower() != mine.lower():
        return ("Mesh road refused: " + str(peer.get("name") or "the peer")
                + " is not a seat of this account.")
    return ""


def _sync_mesh_ask(peer, path, body):
    # One signed sync request; the peer's parsed payload, or None
    # for every failure shape (unlit, refused, misshapen) — the
    # caller treats them all the same: try the next road.
    body = dict(body or {})
    body["requestId"] = uuid.uuid4().hex
    auth = github_auth()
    body["operatorLogin"] = auth["login"] if auth else ""
    # The Mesh wire signs and sends exact bytes (the borrow path
    # passes bytes); the payload is serialized ONCE, here, so the
    # signed bytes are the sent bytes.
    wire = json.dumps(body).encode()
    try:
        status, raw = _mesh_request(peer, "POST", path, wire,
                                     body["requestId"], 120)
    except Exception:
        return None
    if status != 200:
        return None
    try:
        payload = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return None
    if not isinstance(payload, dict) or not payload.get("ok"):
        return None
    return payload


# -- The foreign-data purge (v0.6.4e, #323) -----------------------
# The gate stops new crossings; this cleans up what crossed
# before it existed. Attribution is the evidence: sync_apply
# records, per converged item, the account the arriving set
# named as its own ("" when it named none). On the first gated
# run the standing corpus gets a ledger too — a seat signed in
# adopts its corpus to its own account (the normal upgrade:
# one operator, their machines, their data); a seat NOT signed
# in ledgers "" (unattributed), except day book entries born
# on this very machine, which ledger "local".
#
# Judgment runs only when the seat HAS an account to judge
# against — an unsigned seat purges nothing, ever. An item is
# purgeable when its ledger entry names a DIFFERENT account,
# or names none AND the item carries no local proof. Local
# proof for an unattributed item: a day book entry whose
# origin names this host, or a file whose mtime predates this
# seat's first recorded Mesh sync arrival — material older
# than the first crossing cannot have come through it. With
# NO recorded arrival, unattributed items are NOT purgeable:
# the fail-safe leans toward keeping data, always. Items with
# no ledger entry at all are never touched — they were
# created on this seat after the ledger began.

def _sync_first_mesh_arrival():
    ts = []
    for r in mesh_ledger_read():
        if r.get("direction") == "sync" and r.get("outcome") in ("exchange", "pull"):
            t = r.get("ts")
            if isinstance(t, (int, float)) and not isinstance(t, bool):
                ts.append(t)
    return min(ts) if ts else None


def _purge_predates(path, first_arrival):
    if first_arrival is None:
        return False
    try:
        return os.path.getmtime(path) < first_arrival
    except Exception:
        return False


def sync_account_gate_sweep():
    # The purge judgment. Returns its receipt dict when a
    # judgment ran, else None. Idempotent; safe at bridge
    # start and on sign-in — the two moments the answer can
    # change. Never raises.
    try:
        state = _sync_state_load()
        mine = str((github_auth() or {}).get("login") or "")
        attr = state.get("attribution")
        if not isinstance(attr, dict):
            attr = {"chats": {}, "cases": {}, "docs": {}, "daybook": {}}
            host = socket.gethostname()[:80]
            for c in chats_all():
                attr["chats"][str(c["id"])] = mine
            croot = os.path.join(VAULT_DIR, "cases")
            if os.path.isdir(croot):
                for slug in sorted(os.listdir(croot)):
                    if os.path.isdir(os.path.join(croot, slug)):
                        attr["cases"][str(slug)] = mine
            for name, _label in _SYNC_DOCS:
                if os.path.isfile(_sync_doc_path(name)):
                    attr["docs"][name] = mine
            try:
                for e in _daybook_load().get("entries") or []:
                    oseat = str((e.get("origin") or {}).get("seat") or "")
                    if mine:
                        attr["daybook"][str(e["id"])] = mine
                    else:
                        attr["daybook"][str(e["id"])] = "local" if oseat == host else ""
            except Exception:
                pass
            state["attribution"] = attr
            state["gateLedgerAt"] = int(time.time())
            _sync_state_save(state)
        for fam in ("chats", "cases", "docs", "daybook"):
            if not isinstance(attr.get(fam), dict):
                attr[fam] = {}
        if not mine:
            return None
        foreign_named = any(
            isinstance(v, str) and v and v != "local" and v.lower() != mine.lower()
            for fam in attr.values() if isinstance(fam, dict)
            for v in fam.values())
        if state.get("purgeJudgedFor") == mine and not foreign_named:
            return None
        first_arrival = _sync_first_mesh_arrival()
        host = socket.gethostname()[:80]

        def verdict(fam, key):
            # "purge" / "keep" / None (unattributed — the
            # caller decides on local proof).
            if key not in attr.get(fam, {}):
                return "keep"
            v = attr[fam][key]
            if v == "local":
                return "keep"
            if v:
                return "keep" if str(v).lower() == mine.lower() else "purge"
            return None

        purged = {"chats": [], "cases": [], "docs": [], "daybook": []}
        main_reset = False
        main_held = False
        main_rec = None
        for c in chats_all():
            if str(c["id"]) == "1":
                main_rec = c
        for c in chats_all():
            cid = str(c["id"])
            j = verdict("chats", cid)
            if j is None:
                j = "keep" if _purge_predates(_chat_path(cid), first_arrival) else "purge"
            if j != "purge":
                continue
            if cid == "1":
                # The main thread is never deleted. A main
                # wholly adopted from a foreign set is reset
                # under a fresh clear epoch; a locally-born
                # main with foreign lines merged in cannot be
                # unmerged honestly, so it is HELD and named.
                if _purge_predates(_chat_path("1"), first_arrival):
                    main_held = True
                else:
                    now = time.time()
                    chat_write({"id": "1",
                                "title": (main_rec or {}).get("title") or "Main session",
                                "messages": [],
                                "createdAt": (main_rec or {}).get("createdAt") or now,
                                "updatedAt": now,
                                "clearedAt": now,
                                "clearedBy": _chat_seat_name()})
                    main_reset = True
                continue
            if chat_delete(cid):
                purged["chats"].append(cid)
        croot = os.path.join(VAULT_DIR, "cases")
        if os.path.isdir(croot):
            for slug in sorted(os.listdir(croot)):
                d = os.path.join(croot, slug)
                if not os.path.isdir(d):
                    continue
                j = verdict("cases", slug)
                if j is None:
                    j = "keep" if _purge_predates(d, first_arrival) else "purge"
                if j != "purge":
                    continue
                case_delete(slug)
                if not os.path.isdir(d):
                    purged["cases"].append(slug)
                    legacy = os.path.join(croot, slug + ".md")
                    try:
                        if os.path.isfile(legacy):
                            os.remove(legacy)
                    except Exception:
                        pass
        for name, _label in _SYNC_DOCS:
            j = verdict("docs", name)
            if j is None:
                j = "keep" if _purge_predates(_sync_doc_path(name), first_arrival) else "purge"
            if j != "purge":
                continue
            try:
                if os.path.isfile(_sync_doc_path(name)):
                    os.remove(_sync_doc_path(name))
                    purged["docs"].append(name)
            except Exception:
                pass
        try:
            for e in list(_daybook_load().get("entries") or []):
                eid = str(e["id"])
                j = verdict("daybook", eid)
                if j is None:
                    oseat = str((e.get("origin") or {}).get("seat") or "")
                    j = "keep" if oseat == host else "purge"
                if j != "purge":
                    continue
                gone, _err = daybook_delete_entry(eid)
                if gone is not None:
                    purged["daybook"].append(eid)
        except Exception:
            pass
        for fam, keys in purged.items():
            for k in keys:
                attr.get(fam, {}).pop(k, None)
        if main_reset:
            attr.get("chats", {}).pop("1", None)
        parts = []
        if purged["chats"]:
            parts.append(str(len(purged["chats"])) + " chat" + ("s" if len(purged["chats"]) != 1 else ""))
        if purged["cases"]:
            parts.append(str(len(purged["cases"])) + " case file" + ("s" if len(purged["cases"]) != 1 else ""))
        if purged["docs"]:
            parts.append(str(len(purged["docs"])) + " synced note" + ("s" if len(purged["docs"]) != 1 else ""))
        if purged["daybook"]:
            parts.append(str(len(purged["daybook"])) + " day book entr" + ("ies" if len(purged["daybook"]) != 1 else "y"))
        if main_reset:
            parts.append("the main thread, which had been adopted whole from the foreign set")
        receipt = {"at": int(time.time()), "account": mine,
                   "chats": len(purged["chats"]), "cases": len(purged["cases"]),
                   "docs": len(purged["docs"]), "daybook": len(purged["daybook"]),
                   "mainReset": main_reset, "mainHeld": main_held,
                   "fact": ("Sync's account gate removed material that could not be "
                            "attributed to your account: " + ", ".join(parts) + ".")
                   if parts else ""}
        state = _sync_state_load()
        state["attribution"] = attr
        state["purgeJudgedFor"] = mine
        if parts or main_held:
            state["lastPurge"] = receipt
        _sync_state_save(state)
        return receipt
    except Exception:
        return None


def _sync_mesh_round(cfg, open_chat_id):
    # A finished round result (route named for the peer) when a
    # Mesh exchange completes, else None. Runs under sync_round's
    # lock; the lender serializes its own side the same way.
    peers = _sync_mesh_peers()
    if not peers:
        return None
    collected, manifest, _index = sync_collect()
    if sum(len(b) for b in collected.values()) > _SYNC_MESH_MAX_BYTES:
        return None
    files = {}
    for path, blob in collected.items():
        files[path] = base64.b64encode(blob).decode("ascii")
    local_ids = set(str(c["id"]) for c in chats_payload()["chats"])
    for peer in peers:
        if mesh_probe_peer(peer) is None:
            continue
        refusal = _sync_mesh_peer_gate(peer)
        if refusal:
            _sync_mesh_refusal_note(refusal)
            continue
        payload = _sync_mesh_ask(peer, "/mesh/sync/exchange",
                                 {"files": files, "manifest": manifest})
        if payload is None:
            continue
        theirs = _mesh_sync_decode(payload)
        if theirs is None:
            continue
        if _mesh_sync_foreign_account(payload):
            continue
        _sync_preempt_checkpoint("merge")
        _sync_progress_set("merge")
        summary = sync_apply(cfg, {
            "files": theirs,
            "manifest": payload.get("manifest") if isinstance(payload.get("manifest"), dict) else {},
        })
        # The exchange's first leg parked its own divergences on
        # the LENDER — same operator, same eyes: the borrower's
        # card names them too, or the lender's parked copies
        # would wait for a round that machine never runs.
        applied = payload.get("applied")
        if isinstance(applied, dict):
            for e in (applied.get("eyes") or []):
                if e not in summary["eyes"]:
                    summary["eyes"].append(e)
            for s in (applied.get("caseEyes") or []):
                if s not in summary["caseEyes"]:
                    summary["caseEyes"].append(s)
            summary["changed"] = (summary.get("changed") or 0) + (applied.get("changed") or 0)
        result = _sync_finish_summary(summary, len(files),
                                      open_chat_id, local_ids)
        result["route"] = "Mesh — " + peer["name"]
        result["pushedChats"] = 0
        return result
    return None


def _mesh_sync_foreign_account(payload):
    # The account anchor on the Mesh road (v0.3.0e Build 1,
    # #200; inverted v0.6.4e, #323): a set is applied only when
    # it is ATTRIBUTABLE to this machine's own account — its
    # manifest names an account, this machine knows its own,
    # and the two are equal. Anything else is foreign for this
    # door's purpose: a different account, or a set (or a seat)
    # that cannot be named at all. The handshake is the road's
    # gate; this is the same law read off the set itself, kept
    # as defense in depth against a peer that states one
    # account at the hello and ships another's set.
    manifest = payload.get("manifest") if isinstance(payload, dict) else None
    if not isinstance(manifest, dict):
        return True
    acct = str(manifest.get("account") or "")
    mine = str((github_auth() or {}).get("login") or "")
    if not acct or not mine:
        return True
    return acct.lower() != mine.lower()


def _sync_mesh_pull(cfg):
    # A trigger pull over the Mesh: (route, summary) or None —
    # apply a peer's set, push nothing, mirroring the repo pull.
    for peer in _sync_mesh_peers():
        if mesh_probe_peer(peer) is None:
            continue
        refusal = _sync_mesh_peer_gate(peer)
        if refusal:
            _sync_mesh_refusal_note(refusal)
            continue
        payload = _sync_mesh_ask(peer, "/mesh/sync/pull", {})
        if payload is None:
            continue
        theirs = _mesh_sync_decode(payload)
        if theirs is None:
            continue
        if _mesh_sync_foreign_account(payload):
            continue
        _sync_preempt_checkpoint("merge")
        _sync_progress_set("merge")
        summary = sync_apply(cfg, {
            "files": theirs,
            "manifest": payload.get("manifest") if isinstance(payload.get("manifest"), dict) else {},
        })
        return ("Mesh — " + peer["name"]), summary
    return None


def _sync_progress_set(phase, trigger="", round_id="", counts=None):
    # v0.4.10e (#254): the round's live phase, recorded in the
    # sync state as it happens — fetch, merge, write, done —
    # so the status rail's sync bar can show the phase the
    # bridge is actually in, never a guessed percentage. The
    # write phase carries file counts because the push loop
    # knows them; the fetch leg carries none, because a network
    # wait has no honest fraction. Best-effort by design: a
    # progress write must never fail a round.
    try:
        state = _sync_state_load()
        prog = state.get("syncProgress") if isinstance(state.get("syncProgress"), dict) else {}
        if round_id and prog.get("roundId") != round_id:
            prog = {"roundId": round_id, "startedAt": time.time()}
        prog["active"] = phase != "done"
        prog["phase"] = phase
        prog["phaseAt"] = time.time()
        if trigger:
            prog["trigger"] = trigger
        if counts is not None:
            prog["counts"] = counts
        state["syncProgress"] = prog
        _sync_state_save(state)
    except Exception:
        pass


def _sync_fail_record(trigger, out):
    # v0.4.10e (#254): a failed round lands as a receipt too —
    # the bar names the failure from the record rather than
    # going dark. Mirrors _sync_record_result's save shape.
    try:
        state = _sync_state_load()
        _sync_record_result(state, trigger, out)
        _sync_state_save(state)
    except Exception:
        pass


# -- Push-on-write (v0.5.1e, #257) --------------------------------
# Occasions (the button, app start, a DHD session, Mesh
# contact) cannot meet a ten-second standard by construction:
# a change waits for the other seat's next occasion. So a
# WRITE on this seat asks for a round of its own: the note
# coalesces a burst of writes into one round; the round rides
# the Mesh when a peer's road is lit and the repo otherwise;
# its landing is receipted like any other round's, with the
# road and the measured duration on the bar. No new daemon:
# the note rides the bridge's own life, in a timer thread off
# the write's path. A seat with no road (Mesh dark, signed
# out) notes nothing and names nothing — the occasions carry
# its writes, as they always have. A push that cannot land
# names its stall through the same fail record a failed round
# uses, rather than silently aging.
_SYNC_ROUND = {"depth": 0, "thread": None}
_SYNC_PUSH = {"lock": threading.Lock(), "timer": None,
              "attempts": 0, "kinds": set()}
_SYNC_PUSH_COALESCE = 1.25
_SYNC_PUSH_RETRY = 3.0


# -- Sync preemption (v0.6.3e, #317) ------------------------------
# The override's missing half, named in the field: the operator
# chose Send and the card answered "Held — the running round's
# verdict is the one that counts." An operator's word does not
# queue behind machinery. When a direction is chosen while a
# round is in flight, the choice is NOTED here; the running
# round reads the note at its next phase boundary (the phases
# are already the round's own — fetch, merge, write) and stops
# there, cleanly: rounds converge idempotently, so its partial
# work stands, and its record names the supersession. The
# chosen direction then runs at once. Machine triggers (push-
# on-write, occasions) never note a preemption — the hold stays
# theirs. A newer choice replaces an older one wholesale: the
# newest direction is the one that runs, and a direction
# superseded before it could run says so on its own receipt.


class _SyncSuperseded(Exception):
    # Raised inside a round at a phase boundary when a newer
    # operator direction outranks it. Carries the phase it
    # stopped at and the direction that replaced it.
    def __init__(self, phase, direction):
        super().__init__(phase)
        self.phase = phase
        self.direction = direction


_SYNC_PREEMPT = {"lock": threading.Lock(), "seq": 0, "current": None,
                 "doneSeq": 0, "doneDir": ""}


def _sync_preempt_note(direction, trigger):
    # The operator's choice, recorded. Returns its sequence —
    # the round that runs it holds the sequence so only a NEWER
    # choice can stop it in turn.
    with _SYNC_PREEMPT["lock"]:
        _SYNC_PREEMPT["seq"] += 1
        seq = _SYNC_PREEMPT["seq"]
        _SYNC_PREEMPT["current"] = {"seq": seq, "direction": direction,
                                    "trigger": trigger}
        return seq


def _sync_preempt_checkpoint(phase, own_seq=0):
    # A round's phase boundary: stop here when a newer operator
    # direction stands noted. Legacy and machine rounds hold no
    # sequence (0), so any noted choice stops them; a directional
    # round is stopped only by a newer choice than its own.
    cur = _SYNC_PREEMPT["current"]
    if cur and cur["seq"] > (own_seq or 0):
        raise _SyncSuperseded(phase, cur["direction"])


def _sync_preempt_clear(seq):
    with _SYNC_PREEMPT["lock"]:
        cur = _SYNC_PREEMPT["current"]
        if cur and cur["seq"] == seq:
            _SYNC_PREEMPT["current"] = None


def _sync_superseded_result(direction, phase, own_direction=""):
    # The stopped round's verdict — on its return, its durable
    # record, and the card that reads them back. It names who
    # outranked it and where it stopped; the work it landed
    # before the boundary stands, and says so.
    named = {"send": "Send", "receive": "Receive"}.get(direction, "")
    if own_direction and own_direction != direction:
        message = ("Overridden — a newer choice ("
                   + (named or "a directional round")
                   + ") replaced this "
                   + {"send": "Send", "receive": "Receive"}.get(
                       own_direction, "round")
                   + " at the " + phase + " boundary before it "
                     "finished. Work it had already landed stands.")
    else:
        message = ("Overridden — this round stopped at the "
                   + phase + " boundary: the operator's "
                   + (named or "directional round")
                   + " runs instead. Work it had already landed "
                     "stands; nothing was raced.")
    return {"ok": False, "state": "superseded", "preempted": True,
            "phase": phase, "route": "Backup repo",
            "message": message}


def _sync_preempt_acquire(direction, trigger):
    # A directional round's door (v0.6.3e, #317): note the
    # choice FIRST (the note is the signal the running round
    # reads at its next boundary), then take the round lock —
    # at once when it stands free, or as soon as the stopped
    # (or finished) round releases it. Returns (seq, None) with
    # the lock held, or (seq, result) when a NEWER choice
    # superseded this one while it waited: the newest direction
    # is the one that runs, and this one names its replacement.
    seq = _sync_preempt_note(direction, trigger)
    if _SYNC_LOCK.acquire(False):
        with _SYNC_PREEMPT["lock"]:
            if seq > _SYNC_PREEMPT["doneSeq"]:
                _SYNC_PREEMPT["doneSeq"] = seq
                _SYNC_PREEMPT["doneDir"] = direction
        return seq, None
    _SYNC_LOCK.acquire()
    with _SYNC_PREEMPT["lock"]:
        cur = _SYNC_PREEMPT["current"]
        newer = cur["direction"] if cur and cur["seq"] > seq else None
        dead = newer is not None or seq < _SYNC_PREEMPT["doneSeq"]
        if not dead:
            _SYNC_PREEMPT["doneSeq"] = seq
            _SYNC_PREEMPT["doneDir"] = direction
    if dead:
        _SYNC_LOCK.release()
        out = _sync_superseded_result(
            newer or _SYNC_PREEMPT["doneDir"] or direction,
            "the queue", direction)
        out["direction"] = direction
        return seq, out
    return seq, None


def sync_note_write(kind=""):
    # The write funnels' one call (chat_write, the seat merge,
    # the case mutators, the tombstone record). Best-effort by
    # design: noting a write must never fail the write itself.
    # Writes that happen INSIDE a round, on the round's own
    # thread (convergence landing the other seat's set), note
    # nothing — that round already is the propagation. A write
    # landing on another thread while a round runs is an
    # independent act: it notes, and its push fires behind the
    # round (or retries past it).
    try:
        if _SYNC_ROUND["depth"] > 0 and _SYNC_ROUND.get("thread") == threading.get_ident():
            return
        with _SYNC_PUSH["lock"]:
            _SYNC_PUSH["kinds"].add(str(kind or "write"))
            if _SYNC_PUSH["timer"] is not None:
                return
            t = threading.Timer(_SYNC_PUSH_COALESCE, _sync_push_fire)
            t.daemon = True
            _SYNC_PUSH["timer"] = t
            t.start()
    except Exception:
        pass


def _sync_push_fire():
    with _SYNC_PUSH["lock"]:
        _SYNC_PUSH["timer"] = None
        kinds = set(_SYNC_PUSH["kinds"])
        _SYNC_PUSH["kinds"] = set()
    if not kinds:
        return
    if not _sync_mesh_peers() and not github_auth():
        with _SYNC_PUSH["lock"]:
            _SYNC_PUSH["attempts"] = 0
        return
    result = None
    try:
        result = sync_round(load_config(), "push")
    except Exception:
        result = None
    if isinstance(result, dict) and result.get("ok"):
        with _SYNC_PUSH["lock"]:
            _SYNC_PUSH["attempts"] = 0
        return
    # Any first failure earns exactly one retry: the commonest
    # cause is a collision with a round already in flight on
    # one seat or the other (the lender answers "busy" rather
    # than waiting), and a retry a few seconds later lands
    # behind it. A retry that also fails leaves the rounds'
    # own named receipts standing — except the held shape,
    # which no round records (a busy round returns before
    # recording), so the push names that stall itself.
    with _SYNC_PUSH["lock"]:
        _SYNC_PUSH["attempts"] += 1
        attempts = _SYNC_PUSH["attempts"]
        if attempts < 2:
            _SYNC_PUSH["kinds"] |= kinds
    if attempts < 2:
        with _SYNC_PUSH["lock"]:
            if _SYNC_PUSH["timer"] is None:
                t = threading.Timer(_SYNC_PUSH_RETRY, _sync_push_fire)
                t.daemon = True
                _SYNC_PUSH["timer"] = t
                t.start()
        return
    with _SYNC_PUSH["lock"]:
        _SYNC_PUSH["attempts"] = 0
        _SYNC_PUSH["kinds"] = set()
    held = result is None or (isinstance(result, dict)
                              and (result.get("state") == "held"
                                   or result.get("kind") == "busy"))
    if held:
        _sync_fail_record("push", {
            "ok": False, "state": "held",
            "route": (result or {}).get("route") or "",
            "error": "Push stalled — a sync round was already running and the retry could not land either. The write is safe on this seat; the next occasion carries it."})


def sync_round(cfg, trigger="button", open_chat_id=""):
    # The full round behind the Sync control: pull, converge,
    # push. One round at a time, across every trigger — a second
    # caller is held, never raced.
    if not _SYNC_LOCK.acquire(False):
        return {"ok": False, "state": "held", "kind": "busy", "route": "Backup repo",
                "error": "A sync round is already running — this one is held, and the running round's verdict is the one that counts."}
    round_id = uuid.uuid4().hex[:12]
    if _SYNC_ROUND["depth"] == 0:
        _SYNC_ROUND["thread"] = threading.get_ident()
    _SYNC_ROUND["depth"] += 1
    _round_t0 = time.time()
    _sync_progress_set("fetch", trigger, round_id)
    try:
        # Build 6 (#176 phase 2): a lit Mesh road carries the
        # round before the repo is asked. v0.6.4e (#323): the
        # Mesh road is account-gated — the exchange runs only
        # with a peer the handshake proved a seat of this
        # account, which needs the sign-in the old comment here
        # said it did not. Anything less than a completed
        # exchange falls through to the repo round below, which
        # keeps its own sign-in gate and wording.
        mesh_result = None
        try:
            mesh_result = _sync_mesh_round(cfg, open_chat_id)
        except Exception:
            mesh_result = None
        if mesh_result is not None:
            mesh_result["roundId"] = round_id
            mesh_result["durationMs"] = int((time.time() - _round_t0) * 1000)
            state = _sync_state_load()
            _sync_record_result(state, trigger, mesh_result)
            state["lastRoute"] = mesh_result["route"]
            _sync_state_save(state)
            return mesh_result
        if not github_auth():
            out = _sync_auth_error()
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _round_t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        fetched, fail = sync_fetch(cfg)
        if fail:
            out = {"ok": False, "route": "Backup repo"}
            out.update(fail)
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _round_t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        _sync_preempt_checkpoint("merge")
        _sync_progress_set("merge")
        summary = sync_apply(cfg, fetched)
        _sync_preempt_checkpoint("write")
        got = _sync_repo(cfg)
        if got[0] is None:
            out = {"ok": False, "route": "Backup repo"}
            out.update(got[1] or _sync_auth_error())
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _round_t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        auth, base = got[0]
        local_files, _manifest, _index = sync_collect()
        # What the push changes for the other side, counted in
        # chats: local threads the remote does not hold, or holds
        # older.
        remote_chats = {}
        if SYNC_PREFIX + "chats.json" in fetched["files"]:
            try:
                parsed = json.loads(fetched["files"][SYNC_PREFIX + "chats.json"].decode("utf-8", "replace"))
                if isinstance(parsed, dict):
                    for c in parsed.get("chats") or []:
                        rec = _chat_normalize(c)
                        if rec:
                            remote_chats[str(rec["id"])] = rec
            except Exception:
                remote_chats = {}
        pushed_chats = 0
        local_ids = set()
        for c in chats_payload()["chats"]:
            cid = str(c["id"])
            local_ids.add(cid)
            rc = remote_chats.get(cid)
            if rc is None or (c.get("updatedAt") or 0) > (rc.get("updatedAt") or 0) or _chat_msgs_key(c) != _chat_msgs_key(rc):
                pushed_chats += 1
        written, fail = _sync_push(cfg, base, auth["token"], local_files, fetched["files"])
        if fail:
            out = {"ok": False, "route": "Backup repo"}
            out.update(fail)
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _round_t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        result = _sync_finish_summary(summary, written, open_chat_id, local_ids)
        result["pushedChats"] = pushed_chats
        result["roundId"] = round_id
        result["durationMs"] = int((time.time() - _round_t0) * 1000)
        state = _sync_state_load()
        _sync_record_result(state, trigger, result)
        state["lastRoute"] = "Backup repo"
        _sync_state_save(state)
        return result
    except _SyncSuperseded as e:
        # v0.6.3e (#317): the operator's choice outranked this
        # round at a phase boundary. The stopped round records
        # its own supersession; the direction runs behind it.
        out = _sync_superseded_result(e.direction, e.phase)
        out["roundId"] = round_id
        out["durationMs"] = int((time.time() - _round_t0) * 1000)
        _sync_fail_record(trigger, out)
        return out
    finally:
        _sync_progress_set("done")
        _SYNC_ROUND["depth"] = max(0, _SYNC_ROUND["depth"] - 1)
        if _SYNC_ROUND["depth"] == 0:
            _SYNC_ROUND["thread"] = None
        _SYNC_LOCK.release()


def sync_pull_round(cfg, trigger):
    # The trigger round: pull + converge, no push — the door
    # opening refreshes the room; publishing stays the button's
    # (and the next full round's) act. Quiet by design: verdicts
    # land in the sync state, never in a popup.
    if not _SYNC_LOCK.acquire(False):
        return None
    round_id = uuid.uuid4().hex[:12]
    if _SYNC_ROUND["depth"] == 0:
        _SYNC_ROUND["thread"] = threading.get_ident()
    _SYNC_ROUND["depth"] += 1
    _round_t0 = time.time()
    _sync_progress_set("fetch", trigger, round_id)
    try:
        # Build 6 (#176 phase 2): a lit Mesh road answers a
        # trigger pull before the repo is asked — same quiet
        # shape: apply what arrived, push nothing.
        mesh_hit = None
        try:
            mesh_hit = _sync_mesh_pull(cfg)
        except Exception:
            mesh_hit = None
        if mesh_hit is not None:
            result = _sync_finish_summary(mesh_hit[1], 0, "", set())
            result["route"] = mesh_hit[0]
            result["roundId"] = round_id
            result["durationMs"] = int((time.time() - _round_t0) * 1000)
            state = _sync_state_load()
            _sync_record_result(state, trigger, result)
            state["lastRoute"] = mesh_hit[0]
            _sync_state_save(state)
            return result
        if not github_auth():
            return None
        fetched, fail = sync_fetch(cfg)
        if fail:
            state = _sync_state_load()
            _sync_record_result(state, trigger, {"state": fail.get("state") or "held",
                                                 "message": fail.get("error") or ""})
            return None
        _sync_preempt_checkpoint("merge")
        _sync_progress_set("merge")
        summary = sync_apply(cfg, fetched)
        result = _sync_finish_summary(summary, 0, "", set())
        result["roundId"] = round_id
        result["durationMs"] = int((time.time() - _round_t0) * 1000)
        state = _sync_state_load()
        _sync_record_result(state, trigger, result)
        return result
    except _SyncSuperseded as e:
        # v0.6.3e (#317): even a quiet trigger pull yields to
        # the operator's choice — and names it on the record,
        # though the pull itself stays popup-quiet as ever.
        out = _sync_superseded_result(e.direction, e.phase)
        out["roundId"] = round_id
        out["durationMs"] = int((time.time() - _round_t0) * 1000)
        _sync_fail_record(trigger, out)
        return out
    except Exception:
        return None
    finally:
        _sync_progress_set("done")
        _SYNC_ROUND["depth"] = max(0, _SYNC_ROUND["depth"] - 1)
        if _SYNC_ROUND["depth"] == 0:
            _SYNC_ROUND["thread"] = None
        _SYNC_LOCK.release()


# -- The sync override (v0.6.2e, #303) ---------------------------
# A setting such that hitting the Sync button — the one control,
# wherever it is tapped — offers a choice: Send a sync or
# Receive a sync, a manual override that moves NOW, on demand,
# rather than at the automatic cadence's pace. The directions
# are not a second sync system: Send is the full round's publish
# half standing alone (this seat's set goes out; nothing is
# applied), Receive is the trigger pull with its verdict spoken
# (the converged set comes down; nothing is published). Both
# ride the standing roads — the Mesh first when a peer's road is
# lit, the backup repo otherwise (#257) — and the standing
# convergence law (#251-#253) is untouched. The setting governs
# the button's behavior only: ON (the default — the feature IS
# the behavior), a press offers the choice; OFF, a press runs
# the legacy full round at once. The automatic cadence — app
# start, DHD sessions, Mesh contact, push-on-write — never
# consults the setting.


def sync_choice_enabled(cfg=None):
    # The setting's read: config's sync block, default ON. An
    # absent block, or a value that is not plainly false, reads
    # as the feature standing.
    if cfg is None:
        cfg = load_config()
    try:
        block = cfg.get("sync") if isinstance(cfg, dict) else None
        if isinstance(block, dict) and "choiceOnPress" in block:
            return block.get("choiceOnPress") is not False
    except Exception:
        pass
    return True


def sync_set_choice(on, cfg=None):
    # The setting's stored half: the sync block in the config,
    # written through save_config like every operator setting.
    # Returns the value that now stands, or None when the save
    # itself failed — the caller names that refusal rather than
    # letting the switch pretend.
    if cfg is None:
        cfg = load_config()
    block = cfg.get("sync") if isinstance(cfg.get("sync"), dict) else {}
    block = dict(block)
    block["choiceOnPress"] = bool(on)
    cfg["sync"] = block
    if not save_config(cfg):
        return None
    return bool(on)


def _sync_send_mesh(cfg, files, manifest):
    # Send's Mesh road: the exchange's first leg IS the publish
    # — the peer applies this seat's set at its own door. The
    # set it sends back is deliberately NOT applied here: a
    # Send moves one way. Returns (route, None) on a completed
    # publish, (None, None) when no peer's road is lit.
    peers = _sync_mesh_peers()
    if not peers:
        return None, None
    if sum(len(b) for b in files.values()) > _SYNC_MESH_MAX_BYTES:
        return None, None
    wire = {}
    for path, blob in files.items():
        wire[path] = base64.b64encode(blob).decode("ascii")
    for peer in peers:
        if mesh_probe_peer(peer) is None:
            continue
        refusal = _sync_mesh_peer_gate(peer)
        if refusal:
            _sync_mesh_refusal_note(refusal)
            continue
        payload = _sync_mesh_ask(peer, "/mesh/sync/exchange",
                                 {"files": wire, "manifest": manifest})
        if payload is None:
            continue
        return "Mesh — " + peer["name"], None
    return None, None


_SYNC_STAMP_KEYS = ("exportedAt", "syncedAt", "mtime", "bridgeBuild")


def _sync_substance(path, data):
    # A sync file's substance, with the collect stamps stripped:
    # the set's bookkeeping files restamp themselves on every
    # collect (an exportedAt here, a syncedAt there, a card's
    # mtime), so a byte diff alone would claim movement where
    # nothing of the operator's moved. JSON files compare
    # parsed, with stamp keys dropped at any depth (and the
    # manifest's derived digests, which only echo those
    # stamps); everything else compares byte for byte.
    if data is None:
        return None
    if not str(path).endswith(".json"):
        return data
    try:
        obj = json.loads(data.decode("utf-8", "replace"))
    except Exception:
        return data

    def strip(x, top=False):
        if isinstance(x, dict):
            out = {}
            for k, v in x.items():
                if k in _SYNC_STAMP_KEYS:
                    continue
                if top and str(k).endswith("Sha"):
                    continue
                out[k] = strip(v)
            return out
        if isinstance(x, list):
            return [strip(v) for v in x]
        return x

    try:
        return json.dumps(strip(obj, top=True), sort_keys=True).encode()
    except Exception:
        return data


def sync_send_round(cfg, trigger="button", open_chat_id=""):
    # SEND: publish this seat's set, now — and apply nothing.
    # The receipt names what went up: files written on the repo
    # road; on the Mesh road, the set carried to the named peer.
    # A seat with nothing new to publish says so plainly.
    # v0.6.3e (#317): a Send never queues behind a round in
    # flight — the choice preempts it (see _sync_preempt_acquire).
    _seq, _pre = _sync_preempt_acquire("send", trigger)
    if _pre is not None:
        _pre["roundId"] = uuid.uuid4().hex[:12]
        _sync_fail_record(trigger, _pre)
        return _pre
    round_id = uuid.uuid4().hex[:12]
    if _SYNC_ROUND["depth"] == 0:
        _SYNC_ROUND["thread"] = threading.get_ident()
    _SYNC_ROUND["depth"] += 1
    _t0 = time.time()
    _sync_progress_set("write", trigger, round_id)
    try:
        files, manifest, _index = sync_collect()
        _sync_preempt_checkpoint("write", _seq)
        route, _none = _sync_send_mesh(cfg, files, manifest)
        if route is not None:
            result = {"ok": True, "state": "synced", "direction": "send",
                      "route": route, "filesCarried": len(files),
                      "pushedFiles": len(files), "pushedChats": 0,
                      "message": "Sent — this seat's set ("
                                 + str(len(files)) + " files) went to "
                                 + route + ". Nothing was pulled down."}
            result["roundId"] = round_id
            result["durationMs"] = int((time.time() - _t0) * 1000)
            state = _sync_state_load()
            _sync_record_result(state, trigger, result)
            state["lastRoute"] = route
            _sync_state_save(state)
            return result
        if not github_auth():
            out = _sync_auth_error()
            out["direction"] = "send"
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        got = _sync_repo(cfg)
        if got[0] is None:
            out = {"ok": False, "route": "Backup repo"}
            out.update(got[1] or _sync_auth_error())
            out["direction"] = "send"
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        auth, base = got[0]
        fetched, fail = sync_fetch(cfg)
        if fail:
            out = {"ok": False, "route": "Backup repo"}
            out.update(fail)
            out["direction"] = "send"
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        # What of the operator's actually moves, stamps aside.
        moved = [p for p in files
                 if _sync_substance(p, files[p])
                 != _sync_substance(p, fetched["files"].get(p))]
        if not moved:
            result = {"ok": True, "state": "synced",
                      "direction": "send", "route": "Backup repo",
                      "filesWritten": 0, "pushedFiles": 0,
                      "pushedChats": 0,
                      "message": "Send — nothing to move: this "
                                 "seat's set is already the "
                                 "published set."}
            result["roundId"] = round_id
            result["durationMs"] = int((time.time() - _t0) * 1000)
            state = _sync_state_load()
            _sync_record_result(state, trigger, result)
            state["lastRoute"] = "Backup repo"
            _sync_state_save(state)
            return result
        _sync_preempt_checkpoint("write", _seq)
        written, fail = _sync_push(cfg, base, auth["token"], files,
                                   fetched["files"])
        if fail:
            out = {"ok": False, "route": "Backup repo"}
            out.update(fail)
            out["direction"] = "send"
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        message = ("Sent — " + str(len(moved)) + " file"
                   + ("s" if len(moved) != 1 else "")
                   + " published to the backup repo. Nothing "
                     "was pulled down.")
        result = {"ok": True, "state": "synced", "direction": "send",
                  "route": "Backup repo", "filesWritten": len(moved),
                  "pushedFiles": written, "pushedChats": 0,
                  "message": message}
        result["roundId"] = round_id
        result["durationMs"] = int((time.time() - _t0) * 1000)
        state = _sync_state_load()
        _sync_record_result(state, trigger, result)
        state["lastRoute"] = "Backup repo"
        _sync_state_save(state)
        return result
    except _SyncSuperseded as e:
        # v0.6.3e (#317): a newer operator choice replaced this
        # Send mid-flight; it stops at the boundary and says so.
        out = _sync_superseded_result(e.direction, e.phase, "send")
        out["direction"] = "send"
        out["roundId"] = round_id
        out["durationMs"] = int((time.time() - _t0) * 1000)
        _sync_fail_record(trigger, out)
        return out
    finally:
        _sync_preempt_clear(_seq)
        _sync_progress_set("done")
        _SYNC_ROUND["depth"] = max(0, _SYNC_ROUND["depth"] - 1)
        if _SYNC_ROUND["depth"] == 0:
            _SYNC_ROUND["thread"] = None
        _SYNC_LOCK.release()


def sync_receive_round(cfg, trigger="button"):
    # RECEIVE: pull + converge, publish nothing — the trigger
    # pull's machinery with the button's spoken verdict. The
    # receipt names what came down (the finish summary's own
    # counts); a round where nothing new arrived says so.
    # v0.6.3e (#317): like Send, a Receive preempts a round in
    # flight rather than queueing behind it.
    _seq, _pre = _sync_preempt_acquire("receive", trigger)
    if _pre is not None:
        _pre["roundId"] = uuid.uuid4().hex[:12]
        _sync_fail_record(trigger, _pre)
        return _pre
    round_id = uuid.uuid4().hex[:12]
    if _SYNC_ROUND["depth"] == 0:
        _SYNC_ROUND["thread"] = threading.get_ident()
    _SYNC_ROUND["depth"] += 1
    _t0 = time.time()
    _sync_progress_set("fetch", trigger, round_id)
    try:
        mesh_hit = None
        try:
            mesh_hit = _sync_mesh_pull(cfg)
        except Exception:
            mesh_hit = None
        if mesh_hit is not None:
            result = _sync_finish_summary(mesh_hit[1], 0, "", set())
            result["route"] = mesh_hit[0]
        elif not github_auth():
            out = _sync_auth_error()
            out["direction"] = "receive"
            out["roundId"] = round_id
            out["durationMs"] = int((time.time() - _t0) * 1000)
            _sync_fail_record(trigger, out)
            return out
        else:
            fetched, fail = sync_fetch(cfg)
            if fail:
                out = {"ok": False, "route": "Backup repo"}
                out.update(fail)
                out["direction"] = "receive"
                out["roundId"] = round_id
                out["durationMs"] = int((time.time() - _t0) * 1000)
                _sync_fail_record(trigger, out)
                return out
            _sync_preempt_checkpoint("merge", _seq)
            _sync_progress_set("merge")
            summary = sync_apply(cfg, fetched)
            result = _sync_finish_summary(summary, 0, "", set())
            result["route"] = "Backup repo"
        result["direction"] = "receive"
        if (result.get("state") == "synced"
                and not result.get("receipt", {}).get("changed")
                and not result.get("needsEyes")):
            result["message"] = ("Receive — nothing new came down: "
                                 "this seat already holds the "
                                 "converged set.")
        elif result.get("state") == "synced":
            result["message"] = ("Received — the converged set came "
                                 "down and landed. Nothing was "
                                 "published.")
        result["roundId"] = round_id
        result["durationMs"] = int((time.time() - _t0) * 1000)
        state = _sync_state_load()
        _sync_record_result(state, trigger, result)
        state["lastRoute"] = result["route"]
        _sync_state_save(state)
        return result
    except _SyncSuperseded as e:
        # v0.6.3e (#317): a newer operator choice replaced this
        # Receive mid-flight; it stops at the boundary.
        out = _sync_superseded_result(e.direction, e.phase, "receive")
        out["direction"] = "receive"
        out["roundId"] = round_id
        out["durationMs"] = int((time.time() - _t0) * 1000)
        _sync_fail_record(trigger, out)
        return out
    finally:
        _sync_preempt_clear(_seq)
        _sync_progress_set("done")
        _SYNC_ROUND["depth"] = max(0, _SYNC_ROUND["depth"] - 1)
        if _SYNC_ROUND["depth"] == 0:
            _SYNC_ROUND["thread"] = None
        _SYNC_LOCK.release()


def sync_override_round(cfg, direction, trigger="button",
                        open_chat_id=""):
    # The button's dispatch (#303): a named direction runs its
    # one-way round; anything else is the legacy full round,
    # exactly as the button always ran it.
    if direction == "send":
        return sync_send_round(cfg, trigger, open_chat_id)
    if direction == "receive":
        return sync_receive_round(cfg, trigger)
    return sync_round(cfg, trigger, open_chat_id)


def sync_state_payload():
    auth = github_auth()
    state = _sync_state_load()
    return {
        "signedIn": bool(auth),
        "login": (auth or {}).get("login") or "",
        "route": state.get("lastRoute") or "Backup repo",
        "lastSyncAt": state.get("lastSyncAt"),
        "lastSyncEpoch": state.get("lastSyncEpoch") or 0,
        "lastResult": state.get("lastResult") or "",
        "lastState": state.get("lastState") or "",
        "lastTrigger": state.get("lastTrigger") or "",
        # v0.4.10e (#254): the sync bar's two faces — the live
        # progress while a round runs, the durable receipt
        # after it lands.
        "progress": state.get("syncProgress") or {"active": False, "phase": "done"},
        "receipt": state.get("lastReceipt") or None,
        # v0.5.13e (#262): shells still standing, named — the
        # record clears as fuller arrivals heal them.
        "incomplete": state.get("incomplete") or [],
        # v0.6.2e (#303): the override setting, so every sync
        # surface renders the button's behavior from the bridge.
        "choiceOnPress": sync_choice_enabled(),
        # v0.6.4e (#323): the last Mesh-road account refusal,
        # and the foreign-data purge's standing record.
        "meshRefusal": (state.get("lastReceipt") or {}).get("meshRefusal") or "",
        "lastPurge": state.get("lastPurge") or None,
    }


def _sync_trigger_pull(trigger):
    try:
        cfg = load_config()
        # Build 6: the pull round itself gates the repo path on
        # sign-in; the Mesh path needs none, so triggers no longer
        # pre-gate on the sign-in here.
        sync_pull_round(cfg, trigger)
    except Exception:
        pass


def sync_startup_pull():
    # App start (the __main__ thread beside the backup loop):
    # a short grace for the bridge to finish coming up, then the
    # pull. Inert when signed out.
    time.sleep(10)
    _sync_trigger_pull("app-start")


def _sync_note_session(trigger, key):
    # A session starting at any seat refreshes the room — once,
    # with a global debounce so a door in heavy use pulls at most
    # every debounce window. A device (or window) whose last
    # accepted note is older than the session gap, or which has
    # never been seen, counts as a session starting.
    now = time.time()
    key = str(key or "device")
    with _SYNC_DHD_LOCK:
        last = _SYNC_DHD_SEEN.get(key, 0)
        any_recent = any(now - v < _SYNC_TRIGGER_DEBOUNCE for v in _SYNC_DHD_SEEN.values())
        if now - last < _SYNC_DHD_SESSION_GAP or any_recent:
            if now - last >= _SYNC_DHD_SESSION_GAP:
                _SYNC_DHD_SEEN[key] = now
            return
        _SYNC_DHD_SEEN[key] = now
    threading.Thread(target=_sync_trigger_pull, args=(trigger,), daemon=True).start()


def sync_note_dhd_session(device_key):
    # DHD session start: the proxy calls this for every accepted
    # request from an approved device.
    _sync_note_session("dhd-session", device_key)


def sync_note_app_session(seat_key):
    # v0.3.0e Build 1 (#200): the desktop/app seat notes itself
    # the same way. The bridge is long-lived (boot-armed), so its
    # own app-start pull can be hours stale by the time a window
    # opens; the page notes the seat and the room refreshes for
    # it — the desk side of "pick it up where you left off".
    _sync_note_session("app-session", "seat:" + str(seat_key or "window"))


# --- Boot mode (0.1.9e Build 12, issue #142) -------------------------
# Whether this install is armed to come up headless at boot is read
# from the system every time it is shown — the systemd user units'
# enabled state and the user's linger state — never assumed from
# what an installer once did. Cached briefly: /api/status is polled
# and systemctl is not free.
_BOOT_CACHE = {"at": 0, "payload": None}


def _systemd_user_state(unit):
    try:
        r = subprocess.run(["systemctl", "--user", "is-enabled", unit],
                           capture_output=True, text=True, timeout=3)
        out = (r.stdout or "").strip()
        return out or "unknown"
    except Exception:
        return "unknown"


def _linger_state():
    user = os.environ.get("USER") or ""
    if not user:
        try:
            import pwd
            user = pwd.getpwuid(os.getuid()).pw_name
        except Exception:
            user = ""
    if user:
        try:
            r = subprocess.run(["loginctl", "show-user", user, "--property=Linger", "--value"],
                               capture_output=True, text=True, timeout=3)
            out = (r.stdout or "").strip().lower()
            if out in ("yes", "no"):
                return out == "yes"
        except Exception:
            pass
        try:
            if os.path.exists("/var/lib/systemd/linger/" + user):
                return True
        except Exception:
            pass
    return None


def boot_mode_payload():
    now = time.time()
    if _BOOT_CACHE["payload"] is not None and now - _BOOT_CACHE["at"] < 30:
        return _BOOT_CACHE["payload"]
    bridge_unit = _systemd_user_state("navi-bridge.service")
    llama_unit = _systemd_user_state("navi-llamafile.service")
    linger = _linger_state()
    payload = {
        "bridgeUnit": bridge_unit,
        "llamafileUnit": llama_unit,
        "linger": linger,
        "armed": bridge_unit == "enabled" and llama_unit == "enabled" and linger is True,
        # v0.4.10e (#250): the mode Settings itself holds — the
        # installer's startup choice, kept in the config by the
        # installer and by the Settings → System card alike.
        # The status pane reads it to decide whether a BOOT
        # section belongs there at all: the section follows
        # Settings, and the states inside it still read from
        # the system, never assumed.
        "settingsMode": str((load_config() or {}).get("startup") or ""),
    }
    _BOOT_CACHE["at"] = now
    _BOOT_CACHE["payload"] = payload
    return payload


# --- Arm / disarm at boot from inside the app (Build 14, #159) -----
# The Boot row read the state live from Build 12 on; this is the
# hand that goes with the eyes. Arming does exactly what the
# installer's "At system boot" choice does — install both unit
# files from the templates this package ships (with this install
# folder substituted), daemon-reload, enable --now both units —
# and then tries linger the one way the app honestly can: a plain
# loginctl call, no sudo (the bridge has no sudo channel and does
# not borrow the parked sudo toggle). When linger cannot be
# enabled from here, the result carries the exact command for a
# terminal, the way the installer prints it. Disarming disables
# both units (never --now: what is running keeps running — it
# simply will not come up at boot) and deliberately leaves linger
# alone. Both directions are idempotent, and the verdict is never
# the steps' word: the state is re-read from the system at the
# end and reported as found — partial shows as partial.

_BOOT_UNITS = ("navi-llamafile.service", "navi-bridge.service")


def _boot_user():
    user = os.environ.get("USER") or ""
    if not user:
        try:
            import pwd
            user = pwd.getpwuid(os.getuid()).pw_name
        except Exception:
            user = ""
    return user


def _boot_run(argv, timeout=25):
    try:
        r = subprocess.run(argv, capture_output=True, text=True, timeout=timeout)
        return r.returncode, ((r.stdout or "") + (r.stderr or "")).strip()
    except Exception as e:
        return None, str(e)


def boot_set_armed(want_armed):
    steps = []
    user = _boot_user()
    linger_cmd = ("sudo loginctl enable-linger " + user) if user else None
    unit_dir = os.path.join(os.path.expanduser("~"), ".config", "systemd", "user")
    if want_armed:
        for unit in _BOOT_UNITS:
            try:
                with open(os.path.join(ROOT, "systemd", unit)) as f:
                    text = f.read().replace("__INSTALL_DIR__", ROOT)
                os.makedirs(unit_dir, exist_ok=True)
                with open(os.path.join(unit_dir, unit), "w") as f:
                    f.write(text)
                steps.append({"step": "install " + unit, "ok": True})
            except Exception as e:
                steps.append({"step": "install " + unit, "ok": False, "detail": str(e)})
        rc, out = _boot_run(["systemctl", "--user", "daemon-reload"])
        steps.append({"step": "daemon-reload", "ok": rc == 0, "detail": out if rc != 0 else ""})
        for unit in _BOOT_UNITS:
            rc, out = _boot_run(["systemctl", "--user", "enable", "--now", unit])
            steps.append({"step": "enable " + unit, "ok": rc == 0, "detail": out if rc != 0 else ""})
        if _linger_state() is True:
            steps.append({"step": "linger", "ok": True, "detail": "already on"})
        elif user:
            rc, out = _boot_run(["loginctl", "enable-linger", user])
            steps.append({"step": "linger", "ok": rc == 0, "detail": out if rc != 0 else ""})
        else:
            steps.append({"step": "linger", "ok": False, "detail": "no user name to enable linger for"})
        try:
            cfg = load_config()
            cfg["startup"] = "on-boot"
            save_config(cfg)
        except Exception:
            pass
    else:
        for unit in reversed(_BOOT_UNITS):
            rc, out = _boot_run(["systemctl", "--user", "disable", unit])
            steps.append({"step": "disable " + unit, "ok": rc == 0, "detail": out if rc != 0 else ""})
        _boot_run(["systemctl", "--user", "daemon-reload"])
        try:
            cfg = load_config()
            cfg["startup"] = "on-ui-launch"
            save_config(cfg)
        except Exception:
            pass
    _BOOT_CACHE["at"] = 0
    _BOOT_CACHE["payload"] = None
    boot = boot_mode_payload()
    if want_armed:
        units_on = boot["bridgeUnit"] == "enabled" and boot["llamafileUnit"] == "enabled"
        if boot["armed"]:
            return {"ok": True, "partial": False, "armed": True, "boot": boot,
                    "steps": steps, "lingerCommand": None,
                    "message": "Armed — the system confirms both services are enabled and linger is on. Apex will come up headless at boot."}
        if units_on:
            return {"ok": False, "partial": True, "armed": False, "boot": boot,
                    "steps": steps, "lingerCommand": linger_cmd,
                    "message": "Half-armed: both services are enabled, but linger is still off, so for now they start at login, not at boot. One command in a terminal finishes it:"}
        return {"ok": False, "partial": False, "armed": False, "boot": boot,
                "steps": steps, "lingerCommand": None,
                "message": "Arming did not take — the system does not report the services enabled. The steps above say where it stopped; the installer remains the fallback."}
    still_on = boot["bridgeUnit"] == "enabled" or boot["llamafileUnit"] == "enabled"
    if not still_on:
        return {"ok": True, "partial": False, "armed": False, "boot": boot,
                "steps": steps, "lingerCommand": None,
                "message": "Disarmed — Apex will not come up at boot. What is running now keeps running; linger was left as it is, on purpose."}
    return {"ok": False, "partial": True, "armed": bool(boot["armed"]), "boot": boot,
            "steps": steps, "lingerCommand": None,
            "message": "Disarming only half-took — the system still reports a service enabled. The steps above say which."}


# --- Case Files (0.1.9e Build 12, issues #137/#138) ------------------
# The first Apex | App. A case is a folder of plain markdown in the
# Vault — cases/<slug>/ — so cases ride the Vault's backups and
# sync like every other note on the desk, and open in Obsidian as
# they lie. case.md carries the manifest as frontmatter; findings
# are notes in findings/ with their own frontmatter (severity,
# internal); evidence is whatever lands in evidence/; the timeline
# is one newest-first markdown file. The report (#138) is composed
# from those same files — internal findings are withheld AND
# counted, never silently omitted.

CASE_SEVERITIES = ["critical", "high", "medium", "low", "info"]

# Build 15 (#162): the image evidence a case report embeds.
CASE_IMAGE_MIMES = {
    ".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
    ".gif": "image/gif", ".webp": "image/webp",
}
CASE_EMBED_MAX_BYTES = 8 * 1024 * 1024


def _fm_parse(text):
    meta = {}
    body = text or ""
    if body.startswith("---\n"):
        end = body.find("\n---\n", 3)
        if end != -1:
            head = body[4:end]
            body = body[end + 5:]
            for ln in head.split("\n"):
                if ":" in ln:
                    k, v = ln.split(":", 1)
                    meta[k.strip().lower()] = v.strip()
    return meta, body


def _fm_render(meta_pairs, body):
    lines = ["---"]
    for k, v in meta_pairs:
        lines.append(k + ": " + str(v))
    lines.append("---")
    lines.append("")
    return "\n".join(lines) + body


def _case_slug(name):
    s = re.sub(r"[^A-Za-z0-9]+", "-", str(name or "")).strip("-").lower()
    return s or "case"


def _case_dir(slug):
    if not re.match(r"^[a-z0-9][a-z0-9-]*$", str(slug or "")):
        return None
    rel = vault_join("cases/" + slug)
    if rel is None:
        return None
    return vault_fs_path(rel)


def _case_manifest(slug):
    d = _case_dir(slug)
    if not d or not os.path.isdir(d):
        return None
    try:
        with open(os.path.join(d, "case.md"), encoding="utf-8") as f:
            text = f.read()
    except Exception:
        return None
    meta, _body = _fm_parse(text)
    return meta


def _case_findings(slug):
    d = _case_dir(slug)
    out = []
    fdir = os.path.join(d, "findings") if d else ""
    if fdir and os.path.isdir(fdir):
        for name in sorted(os.listdir(fdir)):
            if not name.lower().endswith(".md"):
                continue
            try:
                with open(os.path.join(fdir, name), encoding="utf-8") as f:
                    text = f.read()
            except Exception:
                continue
            meta, body = _fm_parse(text)
            # v0.6.6e (#329): the read every surface shares —
            # the file's own "# title" line is structure, not
            # body, and comes off; the body's own opening line
            # is the author's text and stands (#291 verbatim).
            # The renderings that print the heading separately
            # (the formed document, the report export) dedupe
            # the opener at their own hand. The stored file is
            # never touched by a read.
            _ttl = str(meta.get("title") or name[:-3])
            _blines = body.split("\n")
            for _i, _ln in enumerate(_blines):
                if not _ln.strip():
                    continue
                if _ln.lstrip().startswith("#") and _opener_norm(_ln) == _opener_norm(_ttl):
                    body = "\n".join(_blines[_i + 1:]).lstrip("\n")
                break
            sev = str(meta.get("severity") or "info").lower()
            if sev not in CASE_SEVERITIES:
                sev = "info"
            out.append({
                "file": name,
                "title": str(meta.get("title") or name[:-3]),
                "severity": sev,
                "internal": str(meta.get("internal") or "").lower() in ("yes", "true", "1"),
                "filed": str(meta.get("filed") or ""),
                # Build 15 (#161): who filed this — Navi's own
                # filings are marked as hers in the frontmatter;
                # notes from before the marking existed are the
                # operator's, which is what they always were.
                "author": "navi" if str(meta.get("author") or "").lower() == "navi" else "operator",
                # Build 2 (#199): the review state. Findings
                # filed before it existed take the default their
                # author implies — the operator's are confirmed,
                # Navi's are leads — which is what they always
                # were in practice.
                "state": (str(meta.get("state") or "").lower()
                          if str(meta.get("state") or "").lower() in ("lead", "confirmed")
                          else ("lead" if str(meta.get("author") or "").lower() == "navi" else "confirmed")),
                # v0.5.9e (#280): the identity stamps a filing
                # was born with — a dossier element's id, its
                # dossier, or the element ids a whole-dossier
                # filing carries.
                # v0.5.15e (#291): the destination the filing
                # declared — a finding unless its frontmatter
                # says it landed as a lead.
                "part": (str(meta.get("part") or "").lower()
                         if str(meta.get("part") or "").lower() in ("lead", "finding")
                         else "finding"),
                "element": str(meta.get("element") or ""),
                "dossierId": str(meta.get("dossier") or ""),
                "elements": [p.strip() for p in str(meta.get("elements") or "").split(",") if p.strip()],
                "body": body.strip(),
            })
    out.sort(key=lambda f: (CASE_SEVERITIES.index(f["severity"]), f["file"]))
    return out


def _sha256_file(path):
    h = hashlib.sha256()
    try:
        with open(path, "rb") as f:
            for chunk in iter(lambda: f.read(1024 * 256), b""):
                h.update(chunk)
        return h.hexdigest()
    except Exception:
        return ""


def _evidence_register_sync(slug):
    # Build 2 (#199): every file in evidence/ gets its custody
    # register entry. Files filed before the register existed
    # are backfilled on first sight — hash computed now and the
    # entry says so honestly (acquired unknown, custody noting
    # the backfill) — so the register the operator reviews is
    # always the whole folder, and Sync carries it whole.
    d = _case_dir(slug)
    if not d:
        return
    edir = os.path.join(d, "evidence")
    if not os.path.isdir(edir):
        return
    have = {e["key"] for e in _case_register(slug, "evidence")}
    now = time.strftime("%Y-%m-%d %H:%M", time.localtime())
    for name in sorted(os.listdir(edir)):
        full = os.path.join(edir, name)
        if not os.path.isfile(full) or _register_key(name) in have:
            continue
        _register_append(slug, "evidence", name, {
            "sha256": _sha256_file(full),
            "size": str(os.path.getsize(full)),
            "acquired": "not recorded — the file predates the evidence register",
            "custody": "Backfilled by the bridge when the register was introduced (v0.3.0e — Build 2); the hash is the file's hash as found.",
            "author": "operator", "filed": now,
        }, "")


def _case_evidence(slug):
    # Files on disk, joined with their custody register entries
    # (description, acquired, custody note, and the SHA-256 the
    # bridge computed at filing time). A file whose entry is
    # missing (a read racing a backfill) still lists, hashless
    # and plainly marked unregistered.
    d = _case_dir(slug)
    out = []
    edir = os.path.join(d, "evidence") if d else ""
    reg = {e["key"]: e for e in _case_register(slug, "evidence")}
    seen = set()
    if edir and os.path.isdir(edir):
        for name in sorted(os.listdir(edir)):
            full = os.path.join(edir, name)
            if os.path.isfile(full):
                e = reg.get(_register_key(name))
                f = e["fields"] if e else {}
                seen.add(_register_key(name))
                out.append({
                    "name": name, "size": os.path.getsize(full),
                    "sha256": str(f.get("sha256") or ""),
                    "description": (e["body"] if e else ""),
                    "acquired": str(f.get("acquired") or ""),
                    "custody": str(f.get("custody") or ""),
                    "registered": e is not None,
                })
    # v0.6.6e (#326): text exhibits — register entries with no
    # file behind them, filed from chat as evidence. They
    # stand in the evidence list as exhibits: the text itself
    # is the item, its custody fields on its face.
    for e in _case_register(slug, "evidence"):
        if e["key"] in seen:
            continue
        f = e["fields"]
        if "text exhibit" not in str(f.get("custody") or "").lower():
            continue
        out.append({
            "name": e["heading"], "size": int(f.get("size") or 0),
            "sha256": str(f.get("sha256") or ""),
            "description": e["body"],
            "acquired": str(f.get("acquired") or ""),
            "custody": str(f.get("custody") or ""),
            "registered": True, "exhibit": True,
        })
    return out


def _case_timeline(slug):
    d = _case_dir(slug)
    out = []
    if not d:
        return out
    try:
        with open(os.path.join(d, "timeline.md"), encoding="utf-8") as f:
            text = f.read()
    except Exception:
        return out
    cur = None
    for ln in text.split("\n"):
        if ln.startswith("## "):
            head = ln[3:].strip()
            date, sep, title = head.partition(" — ")
            cur = {"date": date.strip(), "title": (title if sep else head).strip(), "body": ""}
            out.append(cur)
        elif cur is not None:
            cur["body"] += ln + "\n"
    for e in out:
        e["body"] = e["body"].strip()
    return out


# --- Deep search + the conversational pull (v0.5.10e, #282/#283) --
# The Wire digs deeper, from chat. #282's routes are the #259
# discipline one level down: a named source or a named hit is
# a deterministic route; anything less specific is PROPOSED in
# the open, never wandered into. #283's pull is the intent
# doctrine at the shelf: an operator instruction in a case
# chat has Navi find the card across the Wire's corpora and
# land it on that case's reference shelf, receipted from what
# the bridge actually did — never unbidden.

_DEEP_SOURCE_WORDS = {"nvd": "nvd", "epss": "epss",
                      "exploit-db": "edb", "exploitdb": "edb", "edb": "edb"}


def deep_search_chat_reply(last):
    """#282's deterministic routes. Returns the reply text, or
    None when the words are not a deep ask."""
    text = (last or "").strip()
    if not text:
        return None
    # Move 2 — a named hit to follow outward.
    m = (re.search(r"\b(?:dig|go)\s+deeper\b[^.]*?\b(CVE-\d{4}-\d{4,7})\b", text, re.I)
         or re.search(r"\bdig\s+into\s+(CVE-\d{4}-\d{4,7})\b", text, re.I)
         or re.search(r"\bfollow\s+(CVE-\d{4}-\d{4,7})\b[^.]*?\b(?:outward|out|deeper)\b", text, re.I)
         or re.search(r"\bwhat\s+(?:else|more)\b[^.]*?\b(CVE-\d{4}-\d{4,7})\b", text, re.I))
    if m:
        try:
            return INTEL.deep_follow_text(m.group(1).upper())
        except Exception:
            return ("The dig could not run just now — the store could not "
                    "be read. The Wire page (Apex | Apps → The Wire) shows "
                    "its state.")
    # Move 1 — a named source's own door.
    src, q = None, ""
    m = re.search(r"\b(?:search|check|query|dig(?:\s+into)?)\s+(?:the\s+)?(nvd|epss|exploit-?db|edb)\b[^.]*?\bfor\s+(.+)", text, re.I)
    if m:
        src = _DEEP_SOURCE_WORDS.get(m.group(1).lower().replace(" ", ""), "nvd")
        q = m.group(2)
    else:
        m = re.search(r"\bkev\b[^.]*?\b(?:live|directly|at\s+the\s+source|itself)\b[^.]*?\bfor\s+(.+)", text, re.I)
        if m:
            src, q = "kev", m.group(1)
    if src:
        q = (q or "").strip().rstrip("?").strip()
        if q:
            try:
                return INTEL.deep_search_text(src, q)
            except Exception:
                return ("The source could not be read just now — the Wire "
                        "page (Apex | Apps → The Wire) shows its state.")
    # No door named: propose the dig in the open. Nothing is
    # fetched on a proposal — the fetch waits for a named door.
    if re.search(r"\b(?:dig|search|go)\s+deeper\b", text, re.I) or \
            re.search(r"\bdig\s+into\s+the\s+sources?\b", text, re.I):
        return ("I can dig past the catalog, into the sources themselves — "
                "name the door and I'll go: “search NVD for <terms>”, "
                "“search the KEV live for <terms>”, “check EPSS for "
                "<CVE id>”, or name a record — “dig deeper into CVE-…” — "
                "and I'll follow it outward from its own page (at most six "
                "pages, two hops, every claim cited to its page). Nothing "
                "is fetched until you name the door.")
    return None


def _wire_card_candidates(descriptor):
    # Every card the Wire's corpora can offer for a descriptor,
    # scored: (score, card). The corpora are the ones the Wire
    # page's tabs carry — the Center's catalog, the wire's own
    # items, the CTF board.
    desc = (descriptor or "").strip()
    out = []
    if not desc:
        return out
    ids = intel_mod.CVE_ID_RE.findall(desc) if intel_mod else []
    words = [w for w in re.findall(r"[a-z0-9][a-z0-9._-]{2,}", desc.lower())
             if w not in intel_mod.STOPWORDS] if intel_mod else []

    def score_of(title, body):
        hay = ((title or "") + " " + (body or "")).lower()
        s = 0
        for w in words:
            if w in (title or "").lower():
                s += 3
            elif w in hay:
                s += 1
        return s

    if ids:
        try:
            full = INTEL.item(ids[0].upper())
        except Exception:
            full = None
        if full:
            tab = "kev" if full.get("kev") else "cve"
            text = (full["id"] + " — " + (full.get("title") or "")
                    + "\n" + (full.get("description") or ""))
            out.append((100, {"tab": tab, "title": full["id"] + " — "
                              + (full.get("title") or ""),
                              "link": "https://nvd.nist.gov/vuln/detail/" + full["id"],
                              "source": "CISA KEV" if full.get("kev") else "NVD",
                              "text": text}))
    try:
        res = INTEL.search(desc, limit=5)
        for lite in res.get("results") or []:
            if ids and lite["id"] == ids[0].upper():
                continue
            tab = "kev" if lite.get("kev") else "cve"
            out.append((40 + (10 if lite.get("kev") else 0),
                        {"tab": tab,
                         "title": lite["id"] + " — " + (lite.get("title") or ""),
                         "link": "https://nvd.nist.gov/vuln/detail/" + lite["id"],
                         "source": "CISA KEV" if lite.get("kev") else "NVD",
                         "text": INTEL._lite_line(lite)}))
    except Exception:
        pass
    try:
        for it in (WIRE.state().get("items") or []):
            s = score_of(it.get("title"), it.get("excerpt"))
            if s:
                out.append((s, {"tab": it.get("kind") or "news",
                                "title": it.get("title") or "",
                                "link": it.get("link") or "",
                                "source": it.get("sourceLabel") or "",
                                "text": (it.get("title") or "") + "\n"
                                        + (it.get("sourceLabel") or "")
                                        + "\n\n" + (it.get("excerpt") or "")}))
    except Exception:
        pass
    try:
        st = CTF.state()
        for e in (st.get("running") or []) + (st.get("upcoming") or []):
            s = score_of(e.get("title"), e.get("description"))
            if s:
                paras = [(e.get("format") or "Format unlisted") + " · "
                         + ("On-site — " + (e.get("location") or "place unlisted")
                            if e.get("onsite") else "Online"),
                         (e.get("description") or "")]
                out.append((s, {"tab": "ctfs", "title": e.get("title") or "",
                                "link": e.get("ctftimeUrl") or e.get("url") or "",
                                "source": "CTFtime",
                                "text": (e.get("title") or "") + "\n"
                                        + "\n".join(p for p in paras if p)}))
    except Exception:
        pass
    out.sort(key=lambda t: -t[0])
    return out


def wire_pull_chat_reply(last, chat_id, cfg):
    """#283's conversational pull: “pull that KEV card in”,
    spoken in a case chat (or with a case open), lands the
    card on that case's reference shelf. Returns the reply
    text, or None when the words are not a pull instruction."""
    text = (last or "").strip()
    if not text:
        return None
    m = (re.match(r"(?i)pull\s+(?:in\s+)?(?:that\s+|the\s+)?(CVE-\d{4}-\d{4,7})\b[^.]*?(?:\bcard\b)?[^.]*?(?:\bin\b)?\s*$", text)
         or re.match(r"(?i)pull\s+(?:in\s+)?(?:that\s+|the\s+)?(.+?)\s+cards?\s+in(?:to\s+this\s+case)?\s*$", text)
         or re.match(r"(?i)pull\s+(?:in\s+)?(?:that\s+|the\s+)?(.+?)\s+in(?:to\s+this\s+case)\s*$", text))
    if not m:
        return None
    descriptor = (m.group(1) or "").strip().rstrip(".")
    slug = ""
    if chat_id is not None:
        for rec in chats_all():
            if str(rec.get("id")) == str(chat_id) and rec.get("case"):
                slug = str(rec["case"])
                break
    if not slug:
        slug = case_open_slug(cfg) or ""
    if not slug:
        return ("There's no case room for that pull — open a case (or speak "
                "from its chat) and the card will land on its reference shelf.")
    meta = _case_manifest(slug)
    case_name = str((meta or {}).get("title") or slug)
    cands = _wire_card_candidates(descriptor)
    if not cands:
        return ("No Wire card answers to “" + descriptor[:80] + "” — the "
                "catalog, the wire and the board were all asked. Nothing "
                "was pulled.")
    top_score, top = cands[0]
    tied = [c for s, c in cands[1:] if s >= top_score - 5 and s > 0]
    if tied and not (intel_mod and intel_mod.CVE_ID_RE.search(descriptor)):
        names = ["“" + top["title"][:70] + "”"] + \
                ["“" + c["title"][:70] + "”" for c in tied[:2]]
        return ("More than one card answers to that: " + "; ".join(names)
                + ". Name the one you mean — a title word or its CVE id — "
                "and I'll pull it.")
    ref, note, err = case_shelf_add(slug, top, by="navi")
    if err:
        return "The pull did not land: " + err
    if note == "duplicate":
        return ("“" + (top.get("title") or "")[:90] + "” is already on "
                "this case's shelf — as " + ref["id"] + ". No second copy "
                "was made.")
    return ("Pulled onto the reference shelf of “" + case_name + "” as "
            + ref["id"] + " — “" + (top.get("title") or "")[:90] + "” ("
            + SHELF_TAB_LABELS.get(top.get("tab") or "", top.get("tab") or "")
            + " · " + (top.get("source") or "") + "). It stands as "
            "reference for this case's work — not a finding; the case file "
            "is untouched. Name it (“use reference " + ref["id"] + "”) or "
            "ask me to pull it in a reply and I'll read it in full.")


# --- Case birth from the Wire, and the live briefing -------------
# v0.6.6e (#330, as amended): one act — a button on a Wire
# card, or the words in chat — pulls a story together into a
# case: the article read WHOLE onto the case's reference
# shelf, one cited lead per CVE the article names (resolved
# through the intel catalog; an id the catalog cannot answer
# is a named gap, never a silent skip), related coverage from
# the Wire's other stories shelved beside it, the case set
# open — and the case chat opens on HER BRIEFING, composed
# from the case's live state, with her recommended next tasks
# staged as PROPOSALS on the Tasks register: badged as hers,
# awaiting the operator's word. Approve opens the task;
# dismiss strikes it, the strike on the record. Nothing here
# files a finding: leads are leads, proposals are proposals,
# and the operator's own filings (#326) stay operator acts.

BIRTH_CVE_CAP = 12
BIRTH_RELATED_CAP = 5


def case_stage_task(slug, heading, source):
    """Stage one of Navi's recommended tasks as a proposal.
    Returns the standing entry, or None when it could not be
    staged. A heading already on the register — proposed,
    open or done — is never duplicated."""
    heading = " ".join(str(heading or "").split()).strip()
    if not heading:
        return None
    key = _register_key(heading)
    for e in _case_register(slug, "tasks"):
        if e["key"] == key:
            return e
    now = time.strftime("%Y-%m-%d %H:%M", time.localtime())
    fields = {"done": "no", "state": "proposed", "author": "navi",
              "filed": now, "source": str(source or "recommended by Navi")}
    if not _register_append(slug, "tasks", heading, fields, ""):
        return None
    return {"heading": heading, "key": key, "fields": fields, "body": ""}


def _birth_cve_ids(text):
    ids = []
    if intel_mod:
        for m in intel_mod.CVE_ID_RE.findall(str(text or "")):
            u = m.upper()
            if u not in ids:
                ids.append(u)
    return ids


def _birth_cve_leads(slug, text, source_line):
    """One cited lead per CVE id the text names, resolved
    through the intel catalog. Returns (leads, gaps, overflow):
    leads carry the catalog's own line and description; an id
    the catalog cannot answer lands in gaps — named, never
    silently skipped; ids past the cap land in overflow, also
    named."""
    ids = _birth_cve_ids(text)
    capped, overflow = ids[:BIRTH_CVE_CAP], ids[BIRTH_CVE_CAP:]
    leads, gaps = [], []
    for cid in capped:
        full = None
        try:
            full = INTEL.item(cid)
        except Exception:
            full = None
        if not full:
            gaps.append(cid)
            continue
        try:
            line = INTEL._lite_line(full)
        except Exception:
            line = cid + ((" — " + str(full.get("title"))) if full.get("title") else "")
        body = line
        desc = str(full.get("description") or "").strip()
        if desc:
            body += "\n\n" + desc[:600]
        body += "\n\n— " + source_line
        title = cid + ((" — " + str(full.get("title"))) if full.get("title") else "")
        filed, err = case_land_part(
            slug, "lead", title[:160], body, author="navi",
            source="pulled together at the case's birth from the Wire")
        if err:
            gaps.append(cid)
        else:
            leads.append({"id": cid, "line": line})
    return leads, gaps, overflow


def case_recommended_tasks(slug, leads=None, gaps=None, related=None):
    """Her recommended next tasks for a case, derived from
    what the case actually holds — one per lead (verify it
    against the estate), one per gap (run the id down),
    one for the related coverage (read it for overlap), one
    off the first finding when findings exist. Capped, and
    never duplicating a heading already standing."""
    out = []
    for ld in (leads or []):
        out.append("Verify " + ld["id"] + " against the estate — "
                   "confirm whether anything we run is exposed")
    for g in (gaps or []):
        out.append("Run down " + g + " — the catalog could not "
                   "answer it; pull its record from the source")
    if related:
        out.append("Read the related coverage on the shelf for "
                   "overlap with this case's leads")
    detail = case_detail_payload(slug)
    if detail and detail.get("findings"):
        f0 = detail["findings"][0]
        out.append("Work the first finding: " + str(f0.get("title") or "")[:90])
    if not out:
        out.append("Read the birth article end to end and name "
                   "the first thread to pull")
    return out[:5]


def case_briefing_text(slug):
    """The briefing, composed fresh from the case's live
    state every time it is drawn: what the case was born
    from, what stands on its shelf, its leads and findings,
    its people and machines, its open work — and the tasks
    standing as her proposals. Nothing cached, nothing
    remembered: the case as it stands NOW."""
    detail = case_detail_payload(slug)
    if detail is None:
        return None
    name = str(detail.get("name") or slug)
    lines = ["**Briefing — " + name + "**", ""]
    chat = _case_chat_rec_raw(slug)
    birth = str((chat or {}).get("birthArticle") or "")
    shelf = case_shelf(slug)
    if birth:
        lines.append("Born from the Wire: **" + birth + "**.")
    elif shelf:
        lines.append("Born from the Wire — its first reference "
                     "stands on the shelf.")
    if detail.get("opened"):
        lines.append("Opened " + str(detail["opened"])
                     + " · status " + str(detail.get("status") or "open")
                     + " · type " + str(detail.get("type") or "investigation") + ".")
    lines.append("")
    if shelf:
        lines.append("**On the shelf (" + str(len(shelf)) + "):**")
        for ref in shelf[:8]:
            lines.append("- " + str(ref.get("id") or "") + " — “"
                         + str(ref.get("title") or "")[:100] + "” ("
                         + str(ref.get("source") or "the Wire") + ")")
        lines.append("")
    leads = [f for f in detail.get("findings") or []
             if str(f.get("part") or "") == "lead"
             or str(f.get("state") or "") == "lead"]
    finds = [f for f in detail.get("findings") or [] if f not in leads]
    if leads:
        lines.append("**Leads standing (" + str(len(leads)) + ") — unverified:**")
        for f in leads[:8]:
            lines.append("- " + str(f.get("title") or "")[:130])
        lines.append("")
    if finds:
        lines.append("**Findings (" + str(len(finds)) + "):**")
        for f in finds[:6]:
            lines.append("- " + str(f.get("title") or "")[:130])
        lines.append("")
    for reg, label in (("entities", "Entities"), ("observables", "Observables")):
        ents = detail.get(reg) or []
        if ents:
            lines.append("**" + label + " (" + str(len(ents)) + "):** "
                         + "; ".join(str(e.get("heading") or "")[:60]
                                     for e in ents[:6]))
    tasks = detail.get("tasks") or []
    proposed = [t for t in tasks
                if str((t.get("fields") or {}).get("state") or "") == "proposed"]
    open_tasks = [t for t in tasks
                  if str((t.get("fields") or {}).get("state") or "") != "proposed"
                  and str((t.get("fields") or {}).get("done") or "") != "yes"]
    if open_tasks:
        lines.append("")
        lines.append("**Open tasks (" + str(len(open_tasks)) + "):**")
        for t in open_tasks[:6]:
            lines.append("- " + str(t.get("heading") or "")[:110])
    if proposed:
        lines.append("")
        lines.append("**My recommended next tasks — staged as proposals, "
                     "awaiting your word:**")
        for t in proposed[:6]:
            lines.append("- " + str(t.get("heading") or "")[:110])
        lines.append("")
        lines.append("Approve one and it stands as an open task; dismiss "
                     "one and the strike stands on the record. Both live "
                     "on the case's Tasks register.")
    return "\n".join(lines).strip()


def _chat_append_navi(chat_id, text, extra=None):
    """Append one of her messages to a stored chat — the
    bridge receipt idiom: the room shows it because the
    bridge put it there."""
    for rec in chats_all():
        if str(rec.get("id")) == str(chat_id):
            rec = dict(rec)
            msgs = list(rec.get("messages") or [])
            msg = {"who": "NAVI", "text": text, "exec": None}
            if extra:
                msg.update(extra)
            msgs.append(msg)
            rec["messages"] = msgs
            rec["updatedAt"] = int(time.time() * 1000)
            chat_write(rec)
            return True
    return False

