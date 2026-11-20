# bridge.py — part 10 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 37724–41543 of bridge.py.
Top-level symbols in this part: greeting_payload, _daybook_now_iso, _daybook_day, _daybook_new_id, _daybook_num, _daybook_clean_entry, _daybook_load, _daybook_save, _daybook_get, _daybook_secret_name, daybook_record_capture, daybook_delete_entry, daybook_add_reminder, daybook_postpone, _daybook_day_index, daybook_deferral_count, daybook_quip, daybook_ack, _daybook_clock, _daybook_at_today, _day_parse_when, daybook_parse_order, daybook_add_order, _daybook_report, daybook_execute_order, daybook_tick, daybook_start, _notif_wanted_day, brief_render_md, brief_parse_md, _brief_row, daybook_brief_compose, _daybook_greeting_lines, daybook_payload, daybook_act, _daybook_when_words, day_chat_reply, _sync_apply_daybook, _b64u_dec, _ec_on_curve, _ec_add, _ec_mul, _ec_pub_bytes, _ec_pub_parse, _ec_keypair, _ecdsa_sign, _ecdsa_verify, _ecdh, _hkdf_extract, _hkdf_expand, _aes128_round_keys, _aes_xtime, _aes128_encrypt_block, _ghash_mul, _ghash, _gcm_crypt, _gcm_tag, _aes128gcm_encrypt, _aes128gcm_decrypt, _webpush_derive, _webpush_keys_sender, _webpush_keys_receiver, webpush_encrypt, webpush_decrypt, _webpush_decrypt_with, _b64u_dec_safe, _push_load, _push_save, _push_mint_vapid, push_vapid_public, _push_vapid, _push_vapid_subject, push_vapid_jwt, _push_send_one, _push_note_death, push_deliver_record, push_clear_record, push_subscribe, push_remove_record, push_unsubscribe.


def greeting_payload(cfg=None):
    # The greeting endpoint's answer: a fresh pulse check is
    # folded in first (its freshness window keeps this cheap),
    # then an advancing composition — and the digest it spoke
    # is marked greeted, so the next greeting's brief names
    # only what is newer still.
    if cfg is None:
        cfg = load_config()
    try:
        pulse_maybe_run(cfg=cfg, reason="greeting")
    except Exception:
        pass
    digest = _pulse_digest_lines()
    out = greeting_compose(cfg=cfg, advance=True, digest=digest)
    if digest:
        pulse_mark_greeted([d["batch"] for d in digest])
    return out




# --- Her working day (v0.6.0e, #294/#295/#296) -------------------
# The Assistant Update's foundation: ONE bridge-held store for
# the operator's day — captures, reminders, standing orders and
# the day briefs themselves, as entries in a single file
# (config/daybook.json) behind one lock, one load that cleans,
# one save. The bridge is the truth; every page is a view.
#
# #294 — the foundation store, Capture & Recall built on it.
# The conversational capture machinery is older than this
# store (build .81, #72): the trigger route and the
# [[remember: …]] marker pass decide saves in the bridge, and
# the seat memory (#256) carries them into her context. This
# store is where those decisions are RECORDED — with their
# provenance (which seat, which thread, whose call) — and it
# is a Sync citizen from birth: it rides the account set as
# substance (#262's law), converges by identity with the
# #251–#253 discipline (newest write wins per entry, a
# deletion tombstones by identity and is honored over any
# copy no newer than the deletion), and its saves note a
# push-on-write round like every other funnel's. The store is
# secret-free by construction: the one screen
# (memory_looks_secret, the #72 screen) stands at this store's
# door too, and a credential-shaped capture is refused BY
# NAME — the class of thing it looked like is the refusal.
#
# #295 — reminders & standing orders. A reminder is an entry
# with a time; the scheduler lives HERE, in the resident
# bridge (a tick thread beside Pulse's), its due-state
# persisted in the store across restarts — never in a page's
# timer. Delivery rides the #242 spine: a fired reminder is a
# tray record born through the same reconcile that announces
# it in chat and pushes it to the phone (#272 parity — the
# payload is the record's own summary). Acknowledgment is a
# store write, so it is a SYNCED FACT: acked once, converged
# everywhere, never ringing twice from two seats. A standing
# order is an entry with a trigger and a pre-approved action
# from a BOUNDED vocabulary — remind, report, file,
# watch-check. Sending, spending and elevating are never in
# the vocabulary, at any setting: an order that names one is
# refused by name at the door, and an order that would need
# one asks instead of acting. Every execution lands in the
# order's own execution list — the audit trail a practitioner
# can review after the fact — whether it woke him or not.
#
# #296 — the day brief. The living greeting (#288) grows a
# day section: what is ON, what CHANGED, what NEEDS him, in
# that order, assembled deterministically from what she
# already legitimately sees — this store, Pulse, the case
# list, the tray. NO calendar connector: the brief is built
# from her own records, not a calendar she does not have. A
# source that does not answer is badged as a GAP on the
# brief's face, never silently thinner. The brief is a
# record: it lands in the store, so "what did Tuesday's
# brief say" is a recall question, not a shrug.

DAYBOOK_PATH = os.path.join(ROOT, "config", "daybook.json")
DAYBOOK_KINDS = ("capture", "reminder", "order", "brief")
DAYBOOK_ORDER_VERBS = ("remind", "report", "file", "watch-check")
DAYBOOK_ENTRY_CAP = 400
DAYBOOK_EXEC_CAP = 12
DAYBOOK_TICK_SECONDS = 30
_daybook_lock = threading.Lock()


def _daybook_now_iso(ts=None):
    return time.strftime("%Y-%m-%dT%H:%M:%SZ",
                         time.gmtime(ts or time.time()))


def _daybook_day(ts=None):
    return time.strftime("%Y-%m-%d", time.localtime(ts or time.time()))


def _daybook_new_id(kind, seed):
    base = hashlib.sha256(
        (kind + "|" + str(seed) + "|" + str(time.time())
         + "|" + os.urandom(6).hex()).encode("utf-8")).hexdigest()[:10]
    return {"capture": "cap-", "reminder": "rem-",
            "order": "ord-", "brief": "brief-"}.get(kind, "day-") + base


def _daybook_num(v, default=0):
    return v if isinstance(v, (int, float)) and not isinstance(v, bool) else default


def _daybook_clean_entry(raw):
    # One entry, cleaned — the load's discipline, as everywhere
    # in this bridge: a malformed row is dropped, never
    # repaired into a half-truth the day would then vouch for.
    if not isinstance(raw, dict):
        return None
    kind = str(raw.get("kind") or "").strip()
    eid = str(raw.get("id") or "").strip()
    if kind not in DAYBOOK_KINDS or not eid:
        return None
    out = {
        "id": eid[:80],
        "kind": kind,
        # v0.6.2e (#304): a brief's text is a whole document —
        # its cap is a document's, not a line's.
        "text": str(raw.get("text") or "")[:12000 if kind == "brief" else 1000],
        "createdAt": _daybook_num(raw.get("createdAt")),
        "updatedAt": _daybook_num(raw.get("updatedAt")),
        "origin": dict(raw.get("origin")) if isinstance(raw.get("origin"), dict) else {},
        "state": str(raw.get("state") or "")[:40],
    }
    if kind == "capture":
        out["key"] = str(raw.get("key") or "")[:120] or None
        out["supersedes"] = str(raw.get("supersedes") or "")[:80]
        out["proposed"] = raw.get("proposed") is True
    elif kind == "reminder":
        out["dueAt"] = _daybook_num(raw.get("dueAt"))
        out["firedAt"] = _daybook_num(raw.get("firedAt"))
        out["ackedAt"] = _daybook_num(raw.get("ackedAt"))
        out["orderId"] = str(raw.get("orderId") or "")[:80]
        # v0.6.2e (#304): the entry's act record — postpones
        # live here, and the deferral count derives from them.
        acts = []
        for a in (raw.get("acts") or []):
            if isinstance(a, dict):
                acts.append({
                    "act": str(a.get("act") or "")[:20],
                    "at": _daybook_num(a.get("at")),
                    "from": _daybook_num(a.get("from")),
                    "to": _daybook_num(a.get("to")),
                })
        out["acts"] = acts[-24:]
    elif kind == "order":
        out["verb"] = str(raw.get("verb") or "")[:20]
        trig = raw.get("trigger") if isinstance(raw.get("trigger"), dict) else {}
        out["trigger"] = {
            "kind": str(trig.get("kind") or "brief")[:12],
            "at": _daybook_num(trig.get("at")),
            "every": _daybook_num(trig.get("every")),
        }
        out["params"] = dict(raw.get("params")) if isinstance(raw.get("params"), dict) else {}
        out["wake"] = raw.get("wake") is True
        out["lastRunAt"] = _daybook_num(raw.get("lastRunAt"))
        execs = []
        for e in (raw.get("executions") or []):
            if isinstance(e, dict):
                execs.append({
                    "at": _daybook_num(e.get("at")),
                    "via": str(e.get("via") or "")[:40],
                    "outcome": str(e.get("outcome") or "")[:600],
                    "wake": e.get("wake") is True,
                    "acked": e.get("acked") is True,
                })
        out["executions"] = execs[-DAYBOOK_EXEC_CAP:]
    elif kind == "brief":
        out["day"] = str(raw.get("day") or "")[:10]
        out["via"] = str(raw.get("via") or "")[:20]
        out["lines"] = [str(x)[:500] for x in (raw.get("lines") or [])
                        if str(x or "").strip()][:200]
        out["gaps"] = [str(x)[:200] for x in (raw.get("gaps") or [])
                       if str(x or "").strip()][:8]
        # v0.6.2e (#304): the document itself, and the rows it
        # was composed from, ride the record.
        out["md"] = str(raw.get("md") or raw.get("text") or "")[:12000]
        secs = []
        for sec in (raw.get("sections") or []):
            if not isinstance(sec, dict):
                continue
            rows = []
            for r in (sec.get("rows") or []):
                if isinstance(r, dict):
                    rows.append({
                        "text": str(r.get("text") or "")[:500],
                        "time": str(r.get("time") or "")[:8],
                        "state": str(r.get("state") or "")[:40],
                        "count": int(r.get("count") or 0),
                        "quip": str(r.get("quip") or "")[:500],
                        "gap": r.get("gap") is True,
                        "entryId": str(r.get("entryId") or "")[:80],
                    })
            secs.append({"title": str(sec.get("title") or "")[:40],
                         "rows": rows[:60]})
        out["sections"] = secs[:8]
    return out


def _daybook_load():
    try:
        with open(DAYBOOK_PATH, encoding="utf-8") as f:
            data = json.load(f)
    except Exception:
        data = None
    if not isinstance(data, dict):
        return {"entries": [], "lastBriefDay": "", "lastBriefAt": 0}
    entries = []
    seen = set()
    for raw in (data.get("entries") or []):
        e = _daybook_clean_entry(raw)
        if e is not None and e["id"] not in seen:
            seen.add(e["id"])
            entries.append(e)
    return {"entries": entries[-DAYBOOK_ENTRY_CAP:],
            "lastBriefDay": str(data.get("lastBriefDay") or "")[:10],
            "lastBriefAt": _daybook_num(data.get("lastBriefAt"))}


def _daybook_save(store):
    try:
        os.makedirs(os.path.dirname(DAYBOOK_PATH), exist_ok=True)
        _atomic_write(DAYBOOK_PATH, _json_bytes(store))
    except Exception:
        return False
    sync_note_write("daybook")
    return True


def _daybook_get(store, entry_id):
    for e in store["entries"]:
        if e["id"] == entry_id:
            return e
    return None


def _daybook_secret_name(text):
    # The refusal names the class of thing the screen saw —
    # a refusal that will not say what it refused is a wall,
    # not a discipline. Returns "" when the screen is quiet.
    t = str(text or "")
    if not memory_looks_secret(t):
        return ""
    if MEMORY_LONG_DIGITS_RE.search(t):
        return "a card number"
    low = t.lower()
    if "password" in low or "passcode" in low or "passwd" in low:
        return "a password"
    if "api" in low and "key" in low:
        return "an API key"
    if "ssn" in low or "social security" in low:
        return "a government ID number"
    if "credit card" in low or "card number" in low or "cvv" in low:
        return "card details"
    if "credential" in low:
        return "credentials"
    if "token" in low:
        return "a token"
    return "a secret"


def daybook_record_capture(text, origin="trigger", key=None,
                           proposed=False, chat_id="", cfg=None):
    # The foundation record of a capture. Both conversational
    # doors call this at the moment the bridge receipts the
    # save (the trigger route, the marker pass), and the Day
    # surface's own capture box calls it directly. Returns
    # (entry, error): on a secret-shaped text the error NAMES
    # the class and nothing is written — secret-free by
    # construction, at this door as at every other.
    fact = normalize_memory_fact(text)
    if not fact:
        return None, "Nothing to capture — the line was empty."
    bad = _daybook_secret_name(fact)
    if bad:
        return None, ("Refused: that looks like " + bad
                      + ", and secrets never enter the day book — "
                        "not even when you ask directly.")
    if key is None:
        key = memory_fact_key(fact)
    now = time.time()
    with _daybook_lock:
        store = _daybook_load()
        for e in store["entries"]:
            if (e["kind"] == "capture" and e["state"] == "standing"
                    and not e.get("proposed")
                    and e["text"].lower() == fact.lower()):
                return e, None  # already standing — named, not duplicated
        entry = {
            "id": _daybook_new_id("capture", fact),
            "kind": "capture",
            "text": fact,
            "createdAt": now, "updatedAt": now,
            "origin": {"via": str(origin or "trigger")[:20],
                       "seat": socket.gethostname()[:80],
                       "thread": str(chat_id or "")[:80]},
            "state": "standing",
            "key": key,
            "supersedes": "",
            "proposed": bool(proposed),
        }
        if key:
            # Corrections supersede: a standing capture under
            # the same key is the older statement of the same
            # fact — it steps aside, named, never duplicated.
            for e in store["entries"]:
                if (e["kind"] == "capture" and e["state"] == "standing"
                        and e.get("key") == key
                        and e["text"].lower() != fact.lower()):
                    e["state"] = "superseded"
                    e["updatedAt"] = now
                    entry["supersedes"] = e["id"]
        store["entries"].append(entry)
        _daybook_save(store)
        return entry, None


def daybook_delete_entry(entry_id, cfg=None):
    # Deletion tombstones by identity (the #253 law): the
    # entry goes, the tombstone travels, and no copy no newer
    # than the deletion can resurrect it.
    with _daybook_lock:
        store = _daybook_load()
        e = _daybook_get(store, entry_id)
        if e is None:
            return None, "No such entry stands in the day book."
        store["entries"] = [x for x in store["entries"]
                            if x["id"] != entry_id]
        _daybook_save(store)
    sync_record_tombstone("daybook", entry_id)
    return e, None


def daybook_add_reminder(text, due_at, chat_id="", order_id="",
                         cfg=None):
    text = " ".join(str(text or "").split()).strip()
    if not text:
        return None, "Name what the reminder is for first."
    bad = _daybook_secret_name(text)
    if bad:
        return None, ("Refused: that looks like " + bad
                      + ", and secrets never enter the day book.")
    now = time.time()
    entry = {
        "id": _daybook_new_id("reminder", text),
        "kind": "reminder", "text": text[:300],
        "createdAt": now, "updatedAt": now,
        "origin": {"via": "order" if order_id else "chat",
                   "seat": socket.gethostname()[:80],
                   "thread": str(chat_id or "")[:80]},
        "state": "standing",
        "dueAt": float(due_at), "firedAt": 0, "ackedAt": 0,
        "orderId": str(order_id or "")[:80],
    }
    with _daybook_lock:
        store = _daybook_load()
        store["entries"].append(entry)
        _daybook_save(store)
    return entry, None


# -- Postponement & her quips (v0.6.2e, #304) --------------------
# A reminder is pushed off by an ACT on its own record: the
# entry's acts list gains a postpone entry naming the old and
# new due times, the due time moves, and a fired reminder
# stands again to fire at its new time. The deferral COUNT is
# never stored — it is derived from the record every time it
# is read: the postpone acts on the entry, plus the whole days
# a fired reminder has been carried unacknowledged past the
# day it fired. One source, no counter that can drift.


def daybook_postpone(entry_id, due_at, cfg=None):
    # Postpone the named reminder — or, unnamed, the oldest
    # fired-and-waiting one, else the next standing one. The
    # act lands on the entry's record; the count it feeds is
    # derived, never kept.
    now = time.time()
    with _daybook_lock:
        store = _daybook_load()
        target = None
        if entry_id:
            target = _daybook_get(store, entry_id)
            if target is not None and target["kind"] != "reminder":
                return None, "That entry is not a reminder — only a reminder can be pushed."
        if target is None:
            fired = [e for e in store["entries"]
                     if e["kind"] == "reminder" and e["state"] == "fired"]
            fired.sort(key=lambda e: e.get("firedAt") or e.get("createdAt") or 0)
            standing = [e for e in store["entries"]
                        if e["kind"] == "reminder" and e["state"] == "standing"]
            standing.sort(key=lambda e: e.get("dueAt") or 0)
            target = (fired or standing or [None])[0]
        if target is None:
            return None, "Nothing stands to push — no reminder is waiting or standing."
        acts = target.get("acts")
        if not isinstance(acts, list):
            acts = []
            target["acts"] = acts
        acts.append({"act": "postpone", "at": now,
                     "from": target.get("dueAt") or 0,
                     "to": float(due_at)})
        target["acts"] = acts[-24:]
        target["dueAt"] = float(due_at)
        if target["state"] == "fired":
            # Standing again: the firing is superseded, and
            # the act record above is its history — firedAt
            # means the CURRENT firing, so it clears.
            target["state"] = "standing"
            target["firedAt"] = 0
        target["updatedAt"] = now
        _daybook_save(store)
        return target, None


def _daybook_day_index(epoch):
    lt = time.localtime(epoch or 0)
    return int(time.mktime((lt.tm_year, lt.tm_mon, lt.tm_mday,
                            0, 0, 0, 0, 0, -1)) // 86400)


def daybook_deferral_count(entry, now=None):
    # The count, derived from the entry's own record: one per
    # postpone act, plus one per whole day a fired reminder has
    # been carried past its firing day, unacknowledged.
    now = now or time.time()
    if entry.get("state") in ("acked", "done"):
        return 0
    n = 0
    for a in (entry.get("acts") or []):
        if isinstance(a, dict) and a.get("act") == "postpone":
            n += 1
    if entry.get("state") == "fired" and entry.get("firedAt"):
        n += max(0, _daybook_day_index(now)
                 - _daybook_day_index(entry["firedAt"]))
    return n


# Her voice in the brief, keyed to the count: the second push
# gets a raised eyebrow, the third gets pointed, the fourth
# and beyond get her full attention. Two registers — the
# personal set, and a professional set for items the standing
# orders spawned. Selection is deterministic (the greeting's
# pattern, #288): a stable hash of the entry and the count
# picks the line, so the same count always draws the same
# line, consecutive counts never draw the same line twice in
# a row, and she quips once per count — never again until
# the count moves.
DAYBOOK_QUIPS = {
    2: {
        "personal": (
            "Twice now. I'm not saying anything — I'm just keeping the tally where you can see it.",
            "Pushed again. It doesn't mind. I might.",
            "That's two. The list remembers, even when it's being polite about it.",
        ),
        "pro": (
            "Deferred twice. The record notes it without comment — this is the comment.",
            "Second deferral on this item. Flagging it here so the pattern is visible.",
            "Pushed a second time. Standing items age; the brief is watching the clock on this one.",
        ),
    },
    3: {
        "personal": (
            "Third time this one's moved. It's starting to think you don't like it.",
            "Three pushes. At some point the task becomes the pushing, love.",
            "Pushed a third time. I could keep moving it, or we could admit it's not happening today.",
        ),
        "pro": (
            "Third deferral. Items deferred three times rarely land without a decision — worth one.",
            "This item has been deferred three times. The brief recommends a verdict: schedule it, or strike it.",
            "Three deferrals on record. The pattern, not the item, is what needs attention now.",
        ),
    },
    4: {
        "personal": (
            "Four. FOUR. This reminder has its own forwarding address now.",
            "Pushed again — I've started introducing it to the other furniture. It lives here now.",
            "That's four moves. Either do it, or let me strike it and we'll both stop pretending.",
        ),
        "pro": (
            "Four deferrals. The brief's standing advice: an item deferred this often is a decision being avoided, not a task being scheduled.",
            "Deferred four times. Recommend the operator settle this item's fate at the next review.",
            "Four pushes on record. This item is now the oldest open question on the book.",
        ),
    },
}


def daybook_quip(entry, count=None, now=None):
    # The quip for this entry at this count, or "" — she is
    # silent at counts 0 and 1, and the line is a pure function
    # of (entry, count): once per count, rotating by the count
    # itself so back-to-back counts never repeat a line.
    if count is None:
        count = daybook_deferral_count(entry, now)
    if count < 2:
        return ""
    tier = 4 if count >= 4 else count
    register = "pro" if entry.get("orderId") else "personal"
    lines = DAYBOOK_QUIPS[tier][register]
    seed = 0
    for ch in str(entry.get("id") or entry.get("text") or ""):
        seed = (seed * 31 + ord(ch)) % 997
    return lines[(seed + count) % len(lines)]


def daybook_ack(entry_id=None, cfg=None):
    # Acknowledge the named entry — or, unnamed, the oldest
    # thing standing fired and unacknowledged (a reminder
    # first, then a waking order execution). The ack is a
    # store write: a synced fact, converged everywhere.
    now = time.time()
    with _daybook_lock:
        store = _daybook_load()
        target = None
        if entry_id:
            target = _daybook_get(store, entry_id)
        if target is None:
            fired = [e for e in store["entries"]
                     if e["kind"] == "reminder" and e["state"] == "fired"]
            fired.sort(key=lambda e: e["firedAt"] or e["createdAt"])
            target = fired[0] if fired else None
        if target is not None and target["kind"] == "reminder":
            if target["state"] != "fired":
                return None, "That reminder is not standing fired."
            target["state"] = "acked"
            target["ackedAt"] = now
            target["updatedAt"] = now
            _daybook_save(store)
            # #272 parity, kept on this door too: the ack
            # marks the tray record read through the bell's
            # own write, so the clearing push closes the
            # phone copy — the reconcile that follows retires
            # the record itself.
            try:
                notifications_mark_read(
                    ["reminder:" + target["id"]], cfg=cfg)
            except Exception:
                pass
            return target, None
        for e in store["entries"]:
            if e["kind"] != "order":
                continue
            for x in (e.get("executions") or []):
                if x.get("wake") and not x.get("acked"):
                    x["acked"] = True
                    e["updatedAt"] = now
                    _daybook_save(store)
                    try:
                        notifications_mark_read(
                            ["order:" + e["id"] + ":"
                             + str(int(x.get("at") or 0))], cfg=cfg)
                    except Exception:
                        pass
                    return e, None
        return None, "Nothing is standing fired to acknowledge."


_DAYBOOK_REFUSED_VERBS = {
    "send": "sending", "email": "sending", "mail": "sending",
    "message": "sending", "text": "sending", "post": "sending",
    "reply": "sending", "buy": "spending", "pay": "spending",
    "purchase": "spending", "spend": "spending",
    "elevate": "elevating", "sudo": "elevating",
    "delete": "deleting", "remove": "deleting",
    "run": "running commands", "execute": "executing commands",
    "call": "calling", "order": "ordering goods",
}


def _daybook_clock(text):
    # "8", "8:30", "20:15", with an optional am/pm — one
    # bounded clock shape, read deterministically.
    m = re.match(r"\s*(\d{1,2})(?::(\d{2}))?\s*(am|pm)?\s*$",
                 str(text or ""), re.I)
    if not m:
        return None
    hh = int(m.group(1))
    mm = int(m.group(2) or 0)
    ap = (m.group(3) or "").lower()
    if ap == "pm" and hh < 12:
        hh += 12
    elif ap == "am" and hh == 12:
        hh = 0
    if hh > 23 or mm > 59:
        return None
    return hh, mm


def _daybook_at_today(hh, mm, now=None):
    now = now or time.time()
    lt = time.localtime(now)
    at = time.mktime((lt.tm_year, lt.tm_mon, lt.tm_mday, hh, mm, 0, 0, 0, -1))
    if at <= now:
        at += 86400
    return at


def _day_parse_when(text, now=None):
    # A bounded set of time expressions, read deterministically
    # or not at all — an unreadable time is named and asked,
    # never guessed. Returns (epoch, rest) or (None, text).
    now = now or time.time()
    t = str(text or "")
    m = re.search(r"\bin\s+(\d+)\s*(minutes?|mins?|hours?|hrs?|days?)\b",
                  t, re.I)
    if m:
        n = int(m.group(1))
        unit = m.group(2).lower()
        secs = n * (60 if unit.startswith("min") else
                    3600 if unit.startswith("h") else 86400)
        return now + secs, (t[:m.start()] + t[m.end():]).strip()
    m = re.search(r"\bon\s+(\d{4})-(\d{2})-(\d{2})"
                  r"(?:\s+at\s+(\d{1,2}(?::\d{2})?\s*(?:am|pm)?))?\b",
                  t, re.I)
    if m:
        hh, mm = 9, 0
        if m.group(4):
            c = _daybook_clock(m.group(4))
            if c:
                hh, mm = c
        at = time.mktime((int(m.group(1)), int(m.group(2)),
                          int(m.group(3)), hh, mm, 0, 0, 0, -1))
        return at, (t[:m.start()] + t[m.end():]).strip()
    m = re.search(r"\btomorrow\b(?:\s+at\s+(\d{1,2}(?::\d{2})?\s*(?:am|pm)?))?",
                  t, re.I)
    if m:
        hh, mm = 9, 0
        if m.group(1):
            c = _daybook_clock(m.group(1))
            if c:
                hh, mm = c
        lt = time.localtime(now + 86400)
        at = time.mktime((lt.tm_year, lt.tm_mon, lt.tm_mday,
                          hh, mm, 0, 0, 0, -1))
        return at, (t[:m.start()] + t[m.end():]).strip()
    m = re.search(r"\btonight\b", t, re.I)
    if m:
        return (_daybook_at_today(21, 0, now),
                (t[:m.start()] + t[m.end():]).strip())
    m = re.search(r"\bat\s+(\d{1,2}(?::\d{2})?\s*(?:am|pm)?)\b", t, re.I)
    if m:
        c = _daybook_clock(m.group(1))
        if c:
            return (_daybook_at_today(c[0], c[1], now),
                    (t[:m.start()] + t[m.end():]).strip())
    return None, t


def daybook_parse_order(body, now=None):
    # Parse the action clause of a standing order. Returns
    # (fields, error): the verb must stand in the bounded
    # vocabulary — a verb outside it is refused BY NAME, with
    # the vocabulary named back, at the door (never at
    # execution time, never silently reinterpreted).
    now = now or time.time()
    text = " ".join(str(body or "").split()).strip()
    if not text:
        return None, "The order was empty — name its action first."
    low = text.lower()
    # The operator's natural sentence often leads with the
    # trigger ("every day at 08:00, report the wire count"):
    # the lead clause is read first and supplies the trigger
    # when the action clause carries none of its own.
    lead = ""
    _lm = re.match(
        r"(every\s+day\s+at\s+\d{1,2}:\d{2}|daily\s+at\s+"
        r"\d{1,2}:\d{2}|every\s+\d+\s*(?:hours?|hrs?|minutes?|"
        r"mins?|days?)|when\s+the\s+brief\s+is\s+composed|"
        r"at\s+the\s+brief|in\s+the\s+brief|at\s+\d{1,2}:\d{2})"
        r"\s*,\s*", text, re.I)
    if _lm:
        lead = _lm.group(1)
        text = text[_lm.end():].strip()
        low = text.lower()
    if re.search(r"\bevery\s+(monday|tuesday|wednesday|thursday|"
                 r"friday|saturday|sunday|week)\b", text, re.I):
        return None, ("Weekly orders are not in this cut — name "
                      "it daily, on an interval, at a time, or at "
                      "the brief, and it stands.")
    verb = ""
    rest = text
    m = re.match(r"(watch[-\s]check|remind|report|file)\b", low)
    if m:
        verb = "watch-check" if m.group(1).startswith("watch") else m.group(1)
        rest = text[m.end():].strip()
    else:
        first = low.split(None, 1)[0].strip(".,:") if low.split() else ""
        if first in _DAYBOOK_REFUSED_VERBS:
            return None, ("Refused by name: a standing order cannot "
                          "be a " + _DAYBOOK_REFUSED_VERBS[first]
                          + " order. " + _DAYBOOK_REFUSED_VERBS[first].capitalize()
                          + " is not in the vocabulary — a standing "
                            "order may only remind, report, file, or "
                            "watch-check. I can remind you to do it "
                            "yourself, or report on it; the act itself "
                            "stays yours.")
        return None, ("I can't take that as a standing order — its "
                      "action must open with one of the four verbs: "
                      "remind, report, file, watch-check. Say it as "
                      "\u201cstanding order: report the cases in the "
                      "brief\u201d and it stands.")
    trigger = {"kind": "", "at": 0, "every": 0}
    params = {}
    wake = bool(re.search(r"\b(wake me|tell me|alert me)\b", low))
    m = re.search(r"\bevery\s+(\d+)\s*(hours?|hrs?|minutes?|mins?|days?)\b",
                  rest, re.I)
    if m:
        n = int(m.group(1))
        unit = m.group(2).lower()
        trigger = {"kind": "interval", "at": 0,
                   "every": n * (60 if unit.startswith("min") else
                                 3600 if unit.startswith("h") else 86400)}
        rest = (rest[:m.start()] + rest[m.end():]).strip()
    _dm = re.search(r"\b(?:every\s+day|daily)\s+at\s+"
                    r"(\d{1,2}:\d{2})\b", rest, re.I)
    if _dm and not trigger["kind"]:
        _at, _r = _day_parse_when("at " + _dm.group(1), now)
        if _at:
            trigger = {"kind": "daily", "at": _at, "every": 86400}
            rest = (rest[:_dm.start()] + rest[_dm.end():]).strip()
    elif not trigger["kind"] and re.search(r"\bdaily\b", rest, re.I):
        trigger = {"kind": "interval", "at": 0, "every": 86400}
        rest = re.sub(r"\bdaily\b", "", rest, flags=re.I).strip()
    if not trigger["kind"] and re.search(r"\b(in the|at the) (day )?brief\b",
                                         rest, re.I):
        trigger = {"kind": "brief", "at": 0, "every": 0}
        rest = re.sub(r"\b(in the|at the) (day )?brief\b", "",
                      rest, flags=re.I).strip()
    if not trigger["kind"]:
        at, rest2 = _day_parse_when(rest, now)
        if at:
            trigger = {"kind": "at", "at": at, "every": 0}
            rest = rest2
    if not trigger["kind"] and lead:
        _ll = lead.lower()
        if "brief" in _ll:
            trigger = {"kind": "brief", "at": 0, "every": 0}
        elif re.search(r"every\s+day|daily", _ll):
            _at, _r = _day_parse_when(lead, now)
            if _at:
                trigger = {"kind": "daily", "at": _at, "every": 86400}
        elif _ll.startswith("every"):
            _em = re.search(r"(\d+)\s*(hours?|hrs?|minutes?|mins?|days?)", _ll)
            if _em:
                _n = int(_em.group(1))
                _u = _em.group(2)
                trigger = {"kind": "interval", "at": 0,
                           "every": _n * (60 if _u.startswith("min")
                                          else 3600 if _u.startswith("h")
                                          else 86400)}
        else:
            _at, _r = _day_parse_when(lead, now)
            if _at:
                trigger = {"kind": "at", "at": _at, "every": 0}
    if verb == "remind":
        if not trigger["kind"]:
            return None, ("A remind order needs its time — \u201cat "
                          "18:00\u201d, \u201cevery 4 hours\u201d, "
                          "\u201ctomorrow at 9\u201d. Name it and the "
                          "order stands.")
        params["text"] = re.sub(r"^(me|him|the operator)\b", "",
                                rest, flags=re.I).strip(" .") or text
    elif verb == "report":
        if not trigger["kind"]:
            trigger = {"kind": "brief", "at": 0, "every": 0}
        if re.search(r"\bcase", rest, re.I):
            params["subject"] = "cases"
        elif re.search(r"\bwire\b", rest, re.I):
            params["subject"] = "wire"
        elif re.search(r"\bpulse|watch", rest, re.I):
            params["subject"] = "pulse"
        else:
            params["subject"] = "day"
    elif verb == "file":
        m = re.search(r"\binto\s+(?:the\s+)?(?:case\s+)?"
                      r"([a-z0-9][a-z0-9\-]*)\b", rest, re.I)
        if not m:
            return None, ("A file order names its case — \u201cfile "
                          "<the note> into <case>\u201d. A named case "
                          "that does not exist is named at execution, "
                          "never rerouted.")
        params["case"] = m.group(1)
        note = (rest[:m.start()] + rest[m.end():]).strip(" .")
        note = re.sub(r"^(this|the note|a note)\b", "", note,
                      flags=re.I).strip(" .")
        params["note"] = note or text
        if not trigger["kind"]:
            trigger = {"kind": "brief", "at": 0, "every": 0}
    elif verb == "watch-check":
        topic = re.sub(r"^(on|for)\b", "", rest, flags=re.I).strip(" .")
        params["topic"] = topic
        if not topic:
            return None, ("A watch-check order names its watch — "
                          "\u201cwatch-check on <topic>\u201d.")
        if not trigger["kind"]:
            trigger = {"kind": "interval", "at": 0, "every": 3 * 3600}
    return {"verb": verb, "trigger": trigger, "params": params,
            "wake": wake, "text": text}, None


def daybook_add_order(body, chat_id="", cfg=None):
    fields, err = daybook_parse_order(body)
    if err:
        return None, err
    bad = _daybook_secret_name(fields["text"])
    if bad:
        return None, ("Refused: that order carries " + bad
                      + ", and secrets never enter the day book.")
    now = time.time()
    entry = {
        "id": _daybook_new_id("order", fields["text"]),
        "kind": "order", "text": fields["text"][:600],
        "createdAt": now, "updatedAt": now,
        "origin": {"via": "chat", "seat": socket.gethostname()[:80],
                   "thread": str(chat_id or "")[:80]},
        "state": "standing",
        "verb": fields["verb"], "trigger": fields["trigger"],
        "params": fields["params"], "wake": fields["wake"],
        "lastRunAt": 0, "executions": [],
    }
    with _daybook_lock:
        store = _daybook_load()
        store["entries"].append(entry)
        _daybook_save(store)
    return entry, None


def _daybook_report(subject, cfg):
    # The report verb's whole reach: deterministic summaries
    # of her own records. No model call, nothing leaves the
    # machine, and a source that does not answer says so.
    if subject == "wire":
        try:
            st = WIRE.state() or {}
            items = st.get("items") or []
            news = sum(1 for it in items if it.get("kind") == "news")
            dead = sum(1 for s in (st.get("sources") or [])
                       if s.get("error"))
            return (str(len(items)) + " item(s) on the wire, "
                    + str(news) + " of them news, " + str(dead)
                    + " source(s) not answering")
        except Exception:
            return "the wire did not answer for this report"
    if subject == "cases":
        try:
            heads = cases_list_payload().get("cases") or []
        except Exception:
            return "the case list did not answer for this report"
        if not heads:
            return "no cases stand open"
        parts = []
        for h in heads[:6]:
            parts.append(str(h.get("name") or h.get("slug") or "")
                         + " (" + str(h.get("tasksOpen") or 0)
                         + " task(s) open, "
                         + str(h.get("unverified") or 0)
                         + " unverified)")
        return (str(len(heads)) + " case(s) stand: "
                + "; ".join(parts))
    if subject == "pulse":
        try:
            store = _pulse_load()
        except Exception:
            return "Pulse did not answer for this report"
        if not store["watches"]:
            return "no watches stand"
        return ("watches standing: " + ", ".join(
            "\u201c" + w["topic"] + "\u201d (" + w["state"] + ")"
            for w in store["watches"][:6]))
    try:
        store = _daybook_load()
    except Exception:
        return "the day book did not answer for this report"
    rems = [e for e in store["entries"]
            if e["kind"] == "reminder" and e["state"] in ("standing", "fired")]
    caps = [e for e in store["entries"]
            if e["kind"] == "capture" and e["state"] == "standing"]
    return (str(len(rems)) + " reminder(s) standing, "
            + str(len(caps)) + " capture(s) on the book")


def daybook_execute_order(entry, cfg=None, via="schedule", now=None):
    # One execution, performed deterministically and recorded
    # on the order itself — the receipt is the record, and it
    # stands whether the outcome pleased or not. Returns the
    # outcome line. The caller holds _daybook_lock with the
    # store loaded and saves after.
    now = now or time.time()
    verb = entry.get("verb") or ""
    params = entry.get("params") or {}
    outcome = ""
    if verb == "remind":
        text = str(params.get("text") or entry["text"])
        sub = {
            "id": _daybook_new_id("reminder", text),
            "kind": "reminder", "text": text[:300],
            "createdAt": now, "updatedAt": now,
            "origin": {"via": "order", "seat": socket.gethostname()[:80],
                       "thread": ""},
            "state": "standing", "dueAt": now, "firedAt": 0,
            "ackedAt": 0, "orderId": entry["id"],
        }
        outcome = "reminder set for now: \u201c" + text[:160] + "\u201d"
        entry["_spawn"] = sub
    elif verb == "report":
        outcome = _daybook_report(str(params.get("subject") or "day"), cfg)
    elif verb == "file":
        slug = str(params.get("case") or "")
        note = str(params.get("note") or entry["text"])
        try:
            landed, ferr = case_land_part(
                slug, "finding", "Standing order — " + entry["text"][:80],
                note, author="navi", source="standing-order")
        except Exception as e:
            landed, ferr = None, str(e)[:160]
        if landed:
            outcome = ("filed into " + slug + " at "
                       + str(landed.get("path") or ""))
        else:
            # A named case that does not exist is named, never
            # rerouted — the #291 discipline, kept.
            outcome = ("NOT filed — " + (ferr or "the case did not answer")
                       + " (nothing was rerouted)")
    elif verb == "watch-check":
        topic = str(params.get("topic") or "")
        try:
            pulse_run(cfg=cfg, reason="order")
        except Exception:
            pass
        try:
            store = _pulse_load()
            hit = None
            for w in store["watches"]:
                if w["topic"].lower() == topic.lower():
                    hit = w
                    break
            if hit is None:
                outcome = ("no watch stands on \u201c" + topic
                           + "\u201d — the check ran, the watch did not")
            else:
                outcome = ("checked \u201c" + topic + "\u201d — last "
                           "check " + (hit.get("lastCheckIso") or "never")
                           + ", coverage " + (hit.get("coverage") or "—"))
        except Exception:
            outcome = "Pulse did not answer for this check"
    else:
        outcome = "unknown verb — the order did not act"
    entry.setdefault("executions", []).append({
        "at": now, "via": str(via or "schedule")[:40],
        "outcome": outcome[:600],
        "wake": bool(entry.get("wake")), "acked": False,
    })
    entry["executions"] = entry["executions"][-DAYBOOK_EXEC_CAP:]
    entry["lastRunAt"] = now
    entry["updatedAt"] = now
    return outcome


def daybook_tick(cfg=None, now=None):
    # The scheduler's one pass: fire what is due, run what is
    # owed, open the day's brief at the day's start. Called by
    # the resident loop; the benches call it directly with a
    # fixed clock. Returns what happened, as data.
    if cfg is None:
        cfg = load_config()
    now = now or time.time()
    fired, ran = [], []
    with _daybook_lock:
        store = _daybook_load()
        changed = False
        for e in store["entries"]:
            if (e["kind"] == "reminder" and e["state"] == "standing"
                    and (e.get("dueAt") or 0) <= now):
                e["state"] = "fired"
                e["firedAt"] = now
                e["updatedAt"] = now
                fired.append(e["id"])
                changed = True
        spawns = []
        for e in store["entries"]:
            if e["kind"] != "order" or e["state"] != "standing":
                continue
            trig = e.get("trigger") or {}
            due = False
            if trig.get("kind") == "at" and (trig.get("at") or 0) <= now:
                due = True
            elif trig.get("kind") == "daily" and (trig.get("at") or 0) <= now:
                due = True
            elif trig.get("kind") == "interval" and trig.get("every"):
                last = e.get("lastRunAt") or e.get("createdAt") or 0
                due = (now - last) >= trig["every"]
            if not due:
                continue
            daybook_execute_order(e, cfg, via="schedule", now=now)
            sub = e.pop("_spawn", None)
            if sub is not None:
                spawns.append(sub)
            ran.append(e["id"])
            changed = True
            if trig.get("kind") == "at":
                e["state"] = "done"
            elif trig.get("kind") == "daily":
                _nxt = trig.get("at") or now
                while _nxt <= now:
                    _nxt += 86400
                trig["at"] = _nxt
        # Spawned reminders ride the same store, appended
        # after the pass that made them (never mid-iteration).
        store["entries"].extend(spawns)
        today = _daybook_day(now)
        brief_made = False
        if store.get("lastBriefDay") != today:
            # Only the day is stamped here — the brief's own
            # composition stamps lastBriefAt AFTER reading it,
            # so "captures since the last brief" measures from
            # the brief before this one.
            store["lastBriefDay"] = today
            changed = True
            brief_made = True
        if changed:
            _daybook_save(store)
    if brief_made:
        try:
            daybook_brief_compose(cfg=cfg, via="day-start", now=now)
        except Exception:
            pass
    if fired or ran:
        # Delivery rides the #242 spine: one reconcile births
        # the tray records, announces them in chat, and pushes
        # them to the phone — the same pass every other
        # producer's births ride.
        try:
            notifications_reconcile(cfg)
        except Exception:
            pass
    return {"fired": fired, "ordersRun": ran, "brief": brief_made}


def daybook_start():
    # The resident scheduler, started in __main__ beside
    # Pulse's loop: a first pass shortly after boot (a reminder
    # that fell due while the bridge was down fires on the
    # way back up — due-state is the store's, not a timer's),
    # then a pass every DAYBOOK_TICK_SECONDS.
    def loop():
        time.sleep(45)
        while True:
            try:
                daybook_tick()
            except Exception:
                pass
            time.sleep(DAYBOOK_TICK_SECONDS)
    threading.Thread(target=loop, daemon=True).start()


def _notif_wanted_day(cfg):
    # Tray producers (g) and (h) — #295's delivery on the #242
    # spine. A fired, unacknowledged reminder is wanted until
    # its ack lands in the store (a synced fact — the record
    # retires on every seat when the ack converges). A waking
    # order execution is wanted until it is acknowledged the
    # same way. Births announce in chat and push to the phone
    # by the spine's own rules — parity is inherited, not
    # re-implemented.
    items = []
    try:
        store = _daybook_load()
    except Exception:
        return items
    for e in store["entries"]:
        if e["kind"] == "reminder" and e["state"] == "fired":
            items.append({
                "id": "reminder:" + e["id"],
                "kind": "reminder",
                "title": "Reminder",
                "body": e["text"],
                "target": {"view": "Home"},
                "count": 1,
            })
        elif e["kind"] == "order":
            for x in (e.get("executions") or []):
                if x.get("wake") and not x.get("acked"):
                    items.append({
                        "id": ("order:" + e["id"] + ":"
                               + str(int(x.get("at") or 0))),
                        "kind": "order",
                        "title": "A standing order ran",
                        "body": (e["text"][:200] + " — "
                                 + str(x.get("outcome") or ""))[:600],
                        "target": {"view": "Home"},
                        "count": 1,
                    })
    return items


# -- Brief v2 (v0.6.2e, #304) -------------------------------------
# The brief is composed as a MARKDOWN DOCUMENT and stored as
# one: a dated title, section headings, one bullet per item —
# the bullet journal's discipline. The document is the record:
# recall returns it whole. No surface presents the markdown;
# surfaces DRAW FROM it — the parser below (and its mirror in
# the page) reads the document back into sections and rows,
# and the rows carry their facts in the document's own
# grammar: a leading "HH:MM — " when the item has a time, a
# trailing "[state]" marker where a state matters, and her
# quip as a bullet in her name directly under its item.


def brief_render_md(day, sections, gaps):
    out = ["# Day brief — " + day, ""]
    for title, rows in sections:
        if not rows:
            continue
        out.append("## " + title)
        for r in rows:
            line = "- "
            if r.get("time"):
                line += r["time"] + " — "
            line += r["text"]
            marks = []
            if r.get("state"):
                marks.append(r["state"])
            if (r.get("count") or 0) > 1:
                marks.append("pushed \u00d7" + str(r["count"]))
            if marks:
                line += " [" + " \u00b7 ".join(marks) + "]"
            out.append(line)
            if r.get("quip"):
                out.append("- Navi: " + r["quip"])
        out.append("")
    if gaps:
        out.append("## Gaps")
        for g in gaps:
            out.append("- GAP — " + g
                       + " — the brief names what it could not "
                         "read rather than thinning silently.")
        out.append("")
    return "\n".join(out).rstrip() + "\n"


_BRIEF_ROW_MARKS = re.compile(r"\s\[([^\]]+)\]\s*$")
_BRIEF_ROW_TIME = re.compile(r"^(\d{1,2}:\d{2})\s—\s")


def brief_parse_md(md):
    # The document read back into its list: the one grammar
    # both the bridge and the page parse. Returns
    # {"day": str, "sections": [{"title":..., "rows": [...]}]}
    # where a row is {"text", "time", "state", "count",
    # "quip", "gap"} — a quip bullet folds into the row above
    # it, a GAP bullet stands as its own flagged row.
    doc = {"day": "", "sections": []}
    cur = None
    for raw in str(md or "").splitlines():
        line = raw.rstrip()
        if line.startswith("# "):
            doc["day"] = line[2:].replace("Day brief — ", "").strip()
            continue
        if line.startswith("## "):
            cur = {"title": line[3:].strip(), "rows": []}
            doc["sections"].append(cur)
            continue
        if not line.startswith("- ") or cur is None:
            continue
        body = line[2:].strip()
        if body.startswith("Navi: "):
            if cur["rows"]:
                cur["rows"][-1]["quip"] = body[6:].strip()
            continue
        row = {"text": body, "time": "", "state": "",
               "count": 0, "quip": "", "gap": False}
        if body.startswith("GAP — "):
            row["gap"] = True
            row["text"] = body[6:].strip()
            cur["rows"].append(row)
            continue
        m = _BRIEF_ROW_TIME.match(body)
        if m:
            row["time"] = m.group(1)
            body = body[m.end():]
            row["text"] = body
        m = _BRIEF_ROW_MARKS.search(body)
        if m:
            row["text"] = body[:m.start()].rstrip()
            for part in m.group(1).split(" \u00b7 "):
                part = part.strip()
                if part.startswith("pushed \u00d7"):
                    try:
                        row["count"] = int(part[len("pushed \u00d7"):])
                    except Exception:
                        pass
                elif part:
                    row["state"] = part
        cur["rows"].append(row)
    return doc


def _brief_row(text, entry=None, now=None, state="", when=None):
    row = {"text": text, "time": "", "state": state,
           "count": 0, "quip": "", "gap": False,
           "entryId": (entry or {}).get("id") or ""}
    if when:
        row["time"] = time.strftime("%H:%M", time.localtime(when))
    if entry is not None and entry.get("kind") == "reminder":
        row["count"] = daybook_deferral_count(entry, now)
        if row["count"] > 1:
            row["quip"] = daybook_quip(entry, row["count"], now)
    return row


def daybook_brief_compose(cfg=None, via="on-demand", now=None):
    # #296's composition, #304's form: deterministic, no model
    # call — but the assembly is rows first (one bullet per
    # item, the journal's discipline) and the stored record is
    # the markdown document rendered from them. Sections in
    # the ruled order — On, Changed, Needs you, Standing —
    # and every source wrapped on its own: a source that does
    # not answer is a GAP in the document, never a silent
    # thinning. The brief lands in the store as a record
    # (today's earlier brief is superseded, not stacked).
    if cfg is None:
        cfg = load_config()
    now = now or time.time()
    day = _daybook_day(now)
    # Brief-trigger standing orders execute as the brief is
    # composed — "surface it in the brief" is an instruction
    # with a time, and this is the time. Their outcomes stand
    # on their own records; the sections below then read a
    # store that already carries them.
    try:
        with _daybook_lock:
            _store = _daybook_load()
            _spawns = []
            _ran = False
            for e in _store["entries"]:
                if (e["kind"] == "order" and e["state"] == "standing"
                        and (e.get("trigger") or {}).get("kind") == "brief"):
                    daybook_execute_order(e, cfg, via="brief", now=now)
                    sub = e.pop("_spawn", None)
                    if sub is not None:
                        # Composing twice must not stack twins:
                        # a reminder this order already has
                        # standing (or fired and unacknowledged)
                        # is the same reminder, not a new one.
                        _twin = any(
                            x["kind"] == "reminder"
                            and x.get("orderId") == e["id"]
                            and x["text"] == sub["text"]
                            and x["state"] in ("standing", "fired")
                            for x in _store["entries"] + _spawns)
                        if _twin:
                            if e.get("executions"):
                                e["executions"][-1]["outcome"] = (
                                    e["executions"][-1]["outcome"]
                                    + " (already standing — not "
                                      "duplicated)")[:600]
                        else:
                            _spawns.append(sub)
                    _ran = True
            if _ran:
                _store["entries"].extend(_spawns)
                _daybook_save(_store)
    except Exception:
        pass
    gaps = []
    on, changed, needs, standing_sec = [], [], [], []
    try:
        store = _daybook_load()
        rems = [e for e in store["entries"] if e["kind"] == "reminder"]
        standing = [e for e in rems if e["state"] == "standing"]
        standing.sort(key=lambda e: e.get("dueAt") or 0)
        fired = [e for e in rems if e["state"] == "fired"]
        for e in standing:
            on.append(_brief_row("\u201c" + e["text"][:160] + "\u201d",
                                 e, now, state="standing",
                                 when=e.get("dueAt") or None))
        orders = [e for e in store["entries"]
                  if e["kind"] == "order" and e["state"] == "standing"]
        for o in orders:
            standing_sec.append(_brief_row(
                "\u201c" + o["text"][:120] + "\u201d", None, now,
                state="standing order"))
        last_brief_at = store.get("lastBriefAt") or 0
        fresh_caps = [e for e in store["entries"]
                      if e["kind"] == "capture"
                      and e["state"] == "standing"
                      and (e.get("createdAt") or 0) > last_brief_at]
        for e in fresh_caps[-3:]:
            changed.append(_brief_row(
                "Captured: \u201c" + e["text"][:160] + "\u201d",
                None, now))
        if len(fresh_caps) > 3:
            changed.append(_brief_row(
                "…and " + str(len(fresh_caps) - 3)
                + " more capture(s) since the last brief.", None, now))
        proposed = [e for e in store["entries"]
                    if e["kind"] == "capture" and e.get("proposed")
                    and e["state"] == "standing"]
        for e in proposed:
            needs.append(_brief_row(
                "A capture proposal waits for your word: \u201c"
                + e["text"][:160] + "\u201d", None, now,
                state="waiting"))
        for e in fired:
            needs.append(_brief_row(
                "\u201c" + e["text"][:160] + "\u201d", e, now,
                state="waiting", when=e.get("firedAt") or None))
    except Exception:
        gaps.append("the day book did not answer for this brief")
    try:
        for d in _pulse_digest_lines():
            changed.append(_brief_row("Pulse — " + d["line"], None, now))
    except Exception:
        gaps.append("Pulse did not answer for this brief")
    try:
        heads = cases_list_payload().get("cases") or []
        unv = sum(int(h.get("unverified") or 0) for h in heads
                  if isinstance(h, dict))
        if unv:
            needs.append(_brief_row(
                str(unv) + " unverified case entr"
                + ("y is" if unv == 1 else "ies are")
                + " waiting for your review.", None, now,
                state="waiting"))
        elif heads:
            on.append(_brief_row(
                str(len(heads)) + " case(s) stand, none waiting "
                "on review.", None, now))
    except Exception:
        gaps.append("the case list did not answer for this brief")
    try:
        unread = int(notifications_payload(cfg).get("unread") or 0)
        if unread:
            needs.append(_brief_row(
                str(unread) + " notice(s) stand unread in the tray.",
                None, now, state="waiting"))
    except Exception:
        gaps.append("the tray did not answer for this brief")
    sections = [("On", on), ("Changed", changed),
                ("Needs you", needs), ("Standing", standing_sec)]
    if not any(rows for _t, rows in sections) and not gaps:
        on.append(_brief_row(
            "A quiet day on the book — nothing standing, "
            "nothing waiting.", None, now))
        sections = [("On", on), ("Changed", changed),
                    ("Needs you", needs), ("Standing", standing_sec)]
    md = brief_render_md(day, sections, gaps)
    lines = md.splitlines()
    entry = {
        "id": _daybook_new_id("brief", day + via),
        "kind": "brief", "text": md, "md": md,
        "createdAt": now, "updatedAt": now,
        "origin": {"via": str(via or "on-demand")[:20],
                   "seat": socket.gethostname()[:80], "thread": ""},
        "state": "standing", "day": day, "via": str(via or "on-demand"),
        "lines": lines, "gaps": gaps,
        "sections": [{"title": t, "rows": rows}
                     for t, rows in sections if rows],
    }
    with _daybook_lock:
        store = _daybook_load()
        for e in store["entries"]:
            if (e["kind"] == "brief" and e.get("day") == day
                    and e["state"] == "standing"):
                e["state"] = "superseded"
                e["updatedAt"] = now
        store["entries"].append(entry)
        store["lastBriefDay"] = day
        store["lastBriefAt"] = now
        _daybook_save(store)
    return {"day": day, "via": entry["via"], "lines": lines,
            "gaps": gaps, "text": md, "md": md,
            "sections": entry["sections"], "id": entry["id"]}


def _daybook_greeting_lines(cfg=None):
    # The greeting's day section (#296): the day rides the
    # brief she already speaks — at most two lines, and only
    # when the book actually holds something. An empty book
    # adds nothing (the greeting's shape is untouched).
    try:
        store = _daybook_load()
    except Exception:
        return []
    out = []
    now = time.time()
    fired = [e for e in store["entries"]
             if e["kind"] == "reminder" and e["state"] == "fired"]
    if fired:
        out.append("A reminder is waiting on you: \u201c"
                   + fired[0]["text"][:140] + "\u201d.")
    else:
        standing = [e for e in store["entries"]
                    if e["kind"] == "reminder"
                    and e["state"] == "standing"
                    and _daybook_day(e.get("dueAt") or 0) == _daybook_day(now)]
        if standing:
            standing.sort(key=lambda e: e.get("dueAt") or 0)
            out.append("One reminder stands today — \u201c"
                       + standing[0]["text"][:120] + "\u201d at "
                       + time.strftime("%H:%M", time.localtime(
                           standing[0].get("dueAt") or now)) + ".")
    proposed = [e for e in store["entries"]
                if e["kind"] == "capture" and e.get("proposed")
                and e["state"] == "standing"]
    if proposed:
        out.append(str(len(proposed)) + " capture proposal(s) are "
                   "waiting for your word in the day book.")
    return out[:2]


def daybook_payload(cfg=None):
    # The Day surface's read: the book as it stands — captures
    # (standing, newest first), proposals apart, reminders,
    # orders with their executions, and the latest brief.
    if cfg is None:
        cfg = load_config()
    store = _daybook_load()
    caps = [e for e in store["entries"]
            if e["kind"] == "capture" and e["state"] == "standing"
            and not e.get("proposed")]
    props = [e for e in store["entries"]
             if e["kind"] == "capture" and e["state"] == "standing"
             and e.get("proposed")]
    rems = [e for e in store["entries"] if e["kind"] == "reminder"
            and e["state"] in ("standing", "fired")]
    orders = [e for e in store["entries"] if e["kind"] == "order"
              and e["state"] == "standing"]
    briefs = [e for e in store["entries"] if e["kind"] == "brief"
              and e["state"] == "standing"]
    caps.sort(key=lambda e: -(e.get("createdAt") or 0))
    props.sort(key=lambda e: -(e.get("createdAt") or 0))
    rems.sort(key=lambda e: (0 if e["state"] == "fired" else 1,
                             e.get("dueAt") or 0))
    # v0.6.2e (#304): each reminder row carries its deferral
    # count, derived from its record at read time.
    _now = time.time()
    rems = [dict(e, deferrals=daybook_deferral_count(e, _now))
            for e in rems]
    briefs.sort(key=lambda e: -(e.get("createdAt") or 0))
    return {"ok": True,
            "captures": caps[:30], "proposals": props[:10],
            "reminders": rems[:30], "orders": orders[:20],
            "brief": briefs[0] if briefs else None,
            "lastBriefDay": store.get("lastBriefDay") or ""}


def daybook_act(data, cfg=None):
    # The Day surface's deterministic door — the same
    # machinery the chat routes receipt, never a second
    # implementation of it.
    if cfg is None:
        cfg = load_config()
    data = data or {}
    action = str(data.get("action") or "state")
    if action == "capture":
        entry, err = daybook_record_capture(
            str(data.get("text") or ""), origin="page",
            chat_id=str(data.get("chatId") or ""), cfg=cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "ack":
        entry, err = daybook_ack(str(data.get("id") or "") or None, cfg)
        if err:
            return {"ok": False, "error": err}
        try:
            notifications_reconcile(cfg)
        except Exception:
            pass
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "delete":
        entry, err = daybook_delete_entry(str(data.get("id") or ""), cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "revoke":
        # The Home card's word for the same act the chat door
        # calls a revocation: the entry goes, tombstoned.
        entry, err = daybook_delete_entry(str(data.get("id") or ""), cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "remind":
        due, rest = _day_parse_when(str(data.get("when") or ""))
        if not due:
            return {"ok": False, "error": "Name the time in a shape I "
                    "can read — \u201cin 30 minutes\u201d, \u201cat "
                    "18:30\u201d, \u201ctomorrow at 9\u201d."}
        entry, err = daybook_add_reminder(
            str(data.get("text") or ""), due, cfg=cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "postpone":
        # v0.6.2e (#304): the push-off door — the act lands on
        # the reminder's own record, and the deferral count the
        # brief reads is derived from exactly these acts.
        due, _rest = _day_parse_when(str(data.get("when") or ""))
        if not due:
            return {"ok": False, "error": "Name the new time in a "
                    "shape I can read — \u201cin 30 minutes\u201d, "
                    "\u201cat 18:30\u201d, \u201ctomorrow at 9\u201d."}
        entry, err = daybook_postpone(
            str(data.get("id") or "") or None, due, cfg=cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "order":
        entry, err = daybook_add_order(str(data.get("text") or ""),
                                       cfg=cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    if action == "brief":
        out = daybook_brief_compose(cfg=cfg, via="on-demand")
        return {"ok": True, "brief": out, "day": daybook_payload(cfg)}
    if action == "proposal":
        eid = str(data.get("id") or "")
        with _daybook_lock:
            store = _daybook_load()
            e = _daybook_get(store, eid)
            if e is None or e["kind"] != "capture":
                return {"ok": False, "error": "No such proposal stands."}
            if data.get("accept"):
                e["proposed"] = False
                e["updatedAt"] = time.time()
                _daybook_save(store)
                return {"ok": True, "entry": e,
                        "day": daybook_payload(cfg)}
        entry, err = daybook_delete_entry(eid, cfg)
        if err:
            return {"ok": False, "error": err}
        return {"ok": True, "entry": entry, "day": daybook_payload(cfg)}
    return {"ok": True, "day": daybook_payload(cfg)}


_DAY_REMIND_RE = re.compile(
    r"^\s*remind\s+me\s+(?:to\s+)?(.+?)\s*$", re.I | re.S)
_DAY_ORDER_RES = (
    re.compile(r"^\s*(?:set\s+a\s+)?standing\s+order\s*[:\-]\s*"
               r"(.+?)\s*$", re.I | re.S),
    re.compile(r"^\s*(?:set\s+a\s+)?standing\s+order\s+"
               r"((?:remind|report|file|watch[-\s]check)\b.+?)\s*$",
               re.I | re.S),
)
_DAY_POSTPONE_RE = re.compile(
    r"^\s*(?:snooze(?:\s+(?:it|that|this))?|"
    r"push\s+(?:it|that|this)(?:\s+(?:to|until|off))?)"
    r"\s*(?:to|until|for)?\s*(.*?)\s*$",
    re.I | re.S)
_DAY_ACK_RE = re.compile(
    r"^\s*(got\s+it|ack|acknowledge|acknowledged|"
    r"that'?s\s+handled|handled|done)\s*[.!]?\s*$", re.I)
_DAY_BRIEF_RE = re.compile(
    r"^\s*(brief\s+me|day\s+brief|give\s+me\s+(the|my)\s+brief|"
    r"the\s+day\s+brief)\b", re.I)
_DAY_RECALL_RE = re.compile(
    r"^\s*(?:recall|what\s+do\s+you\s+remember\s+about|"
    r"what\s+did\s+you\s+capture\s+about)\s+(.+?)\s*[.?\s]*$", re.I)
_DAY_UNDO_RE = re.compile(
    r"^\s*(undo\s+(that|it|the\s+capture|that\s+capture)|"
    r"forget\s+that\s+capture)\s*[.!]?\s*$", re.I)
_DAY_ORDERS_LIST_RE = re.compile(
    r"^\s*(list|show)?\s*(my\s+|the\s+)?standing\s+orders\s*[.!?]?\s*$",
    re.I)
_DAY_REVOKE_RE = re.compile(
    r"^\s*(revoke|cancel|drop)\s+(the\s+)?standing\s+order\s+"
    r"(?:on|about|to|for)?\s*(.+?)\s*$", re.I)


def _daybook_when_words(epoch, now=None):
    now = now or time.time()
    lt = time.localtime(epoch)
    if _daybook_day(epoch) == _daybook_day(now):
        return "today at " + time.strftime("%H:%M", lt)
    if _daybook_day(epoch) == _daybook_day(now + 86400):
        return "tomorrow at " + time.strftime("%H:%M", lt)
    return time.strftime("%Y-%m-%d %H:%M", lt)


def day_chat_reply(last, cfg=None, chat_id=""):
    # The working day's conversational door (the intent
    # doctrine): deterministic, ahead of the model, receipted
    # from what the bridge actually did. Every pattern is
    # bounded — a line that is merely TALK about reminders or
    # orders falls through to her, untouched.
    if cfg is None:
        cfg = load_config()
    text = str(last or "").strip()
    if not text:
        return None
    if _DAY_ORDERS_LIST_RE.match(text):
        store = _daybook_load()
        orders = [e for e in store["entries"]
                  if e["kind"] == "order" and e["state"] == "standing"]
        if not orders:
            return ("No standing orders on the book. Set one: "
                    "\u201cstanding order: report the cases in the "
                    "brief\u201d — the vocabulary is remind, report, "
                    "file, watch-check.")
        return ("Standing orders: " + "; ".join(
            "\u201c" + o["text"] + "\u201d (" + o["verb"] + ", "
            + o["trigger"]["kind"] + ")" for o in orders) + ".")
    m = None
    for rx in _DAY_ORDER_RES:
        m = rx.match(text)
        if m:
            break
    if m:
        entry, err = daybook_add_order(m.group(1), chat_id=chat_id,
                                       cfg=cfg)
        if err:
            return err
        trig = entry["trigger"]
        if trig["kind"] == "at":
            when = "at " + _daybook_when_words(trig["at"])
        elif trig["kind"] == "interval" and trig["every"] >= 3600:
            when = ("every " + str(int(trig["every"] // 3600))
                    + " hour(s)")
        elif trig["kind"] == "interval":
            when = ("every " + str(int(trig["every"] // 60))
                    + " minute(s)")
        elif trig["kind"] == "brief":
            when = "in the day brief"
        elif trig["kind"] == "daily":
            when = ("every day at "
                    + time.strftime("%H:%M",
                                    time.localtime(trig.get("at") or 0)))
        else:
            when = "on its trigger"
        return ("Standing order on the book: \u201c" + entry["text"]
                + "\u201d — it will " + entry["verb"] + " " + when
                + ", and every run lands on its record whether it "
                  "wakes you or not. Revoke it any time: "
                  "\u201crevoke the standing order on "
                + entry["text"][:40] + "\u201d.")
    m = _DAY_REVOKE_RE.match(text)
    if m:
        want = m.group(3).lower()
        store = _daybook_load()
        hit = None
        for e in store["entries"]:
            if (e["kind"] == "order" and e["state"] == "standing"
                    and (want in e["text"].lower()
                         or want in e["id"].lower())):
                hit = e
                break
        if hit is None:
            return ("No standing order matches \u201c" + m.group(3)
                    + "\u201d — the book holds: "
                    + (", ".join("\u201c" + o["text"][:50] + "\u201d"
                                 for o in store["entries"]
                                 if o["kind"] == "order"
                                 and o["state"] == "standing")
                       or "none") + ".")
        entry, err = daybook_delete_entry(hit["id"], cfg)
        if err:
            return err
        return ("Revoked — \u201c" + hit["text"]
                + "\u201d is off the book, tombstoned on every "
                  "seat.")
    _rm = re.match(
        r"^\s*remember\s+(?!when\b|me\b|us\b|the\s+time\b)"
        r"(?:that\s+)?(.+?)\s*$", text, re.I | re.S)
    if _rm and not text.rstrip().endswith("?"):
        entry, err = daybook_record_capture(
            _rm.group(1), origin="trigger", chat_id=chat_id, cfg=cfg)
        if err:
            return err
        if entry is None:
            return ("Already on the book — \u201c"
                    + " ".join(_rm.group(1).split())[:160]
                    + "\u201d stands as captured.")
        return ("Kept — \u201c" + entry["text"][:160]
                + "\u201d is in the day book, with its provenance "
                  "on it. \u201cUndo that\u201d takes it back.")
    m = _DAY_REMIND_RE.match(text)
    if m:
        due, rest = _day_parse_when(m.group(1))
        if not due:
            return ("I can't read a time in that — say it as "
                    "\u201cremind me in 30 minutes to …\u201d, "
                    "\u201cremind me at 18:30 to …\u201d, or "
                    "\u201cremind me tomorrow at 9 to …\u201d and it "
                    "stands. Nothing was set.")
        what = re.sub(r"^(to\s+)", "", rest.strip(), flags=re.I)
        what = " ".join(what.split()).strip(" .")
        if not what:
            return "Remind you of what? Name it and it stands."
        entry, err = daybook_add_reminder(what, due,
                                          chat_id=chat_id, cfg=cfg)
        if err:
            return err
        return ("Reminder standing: \u201c" + entry["text"]
                + "\u201d — " + _daybook_when_words(due)
                + ". It fires on whichever seat you're at, once, "
                  "and your \u201cgot it\u201d anywhere settles it "
                  "everywhere.")
    m = _DAY_POSTPONE_RE.match(text)
    if m:
        tail = (m.group(1) or "").strip()
        due = None
        if tail:
            due, _rest = _day_parse_when(tail)
        if due is None and tail in ("", "later"):
            due = time.time() + 3600
        if due is None:
            return ("I can't read a time in that — say "
                    "\u201csnooze it to tomorrow\u201d, "
                    "\u201cpush it in 30 minutes\u201d, or name a "
                    "clock time, and it moves. Nothing moved.")
        entry, err = daybook_postpone(None, due, cfg=cfg)
        if err:
            return err
        reply = ("Pushed — \u201c" + entry["text"][:160]
                 + "\u201d now stands "
                 + _daybook_when_words(due) + ".")
        quip = daybook_quip(entry)
        if quip:
            reply += "\n" + quip
        return reply
    if _DAY_ACK_RE.match(text):
        entry, err = daybook_ack(None, cfg)
        if entry is not None:
            try:
                notifications_reconcile(cfg)
            except Exception:
                pass
            if entry["kind"] == "reminder":
                return ("Settled — \u201c" + entry["text"]
                        + "\u201d is acknowledged, here and on every "
                          "seat.")
            return ("Settled — the standing order's run is "
                    "acknowledged: \u201c" + entry["text"][:120]
                    + "\u201d.")
        return None  # nothing fired — the line is talk, hers to answer
    if _DAY_BRIEF_RE.match(text):
        out = daybook_brief_compose(cfg=cfg, via="on-demand")
        # v0.6.2e (#304): the answer IS the document — the
        # page draws the list from it; the markdown is what
        # the thread holds and what recall returns.
        return out["md"]
    m = _DAY_RECALL_RE.match(text)
    if m:
        want = m.group(1).lower()
        store = _daybook_load()
        hits = [e for e in store["entries"]
                if e["kind"] == "capture"
                and want in e["text"].lower()]
        if not hits:
            return ("Nothing in the day book answers to \u201c"
                    + m.group(1) + "\u201d — the book holds "
                    + str(sum(1 for e in store["entries"]
                              if e["kind"] == "capture"))
                    + " capture(s).")
        hits.sort(key=lambda e: -(e.get("createdAt") or 0))
        rows = []
        for e in hits[:5]:
            rows.append("\u201c" + e["text"] + "\u201d — captured "
                        + _daybook_day(e.get("createdAt") or 0)
                        + ", " + str((e.get("origin") or {}).get("via")
                                     or "chat")
                        + (" (superseded)" if e["state"] == "superseded"
                           else ""))
        return "From the day book: " + "; ".join(rows) + "."
    if _DAY_UNDO_RE.match(text):
        store = _daybook_load()
        caps = [e for e in store["entries"]
                if e["kind"] == "capture" and e["state"] == "standing"]
        if not caps:
            return "No capture stands to undo — the book is clean."
        caps.sort(key=lambda e: -(e.get("createdAt") or 0))
        entry, err = daybook_delete_entry(caps[0]["id"], cfg)
        if err:
            return err
        return ("Undone — \u201c" + caps[0]["text"]
                + "\u201d is out of the day book, on every seat.")
    return None


def _sync_apply_daybook(fetched, summary, tombs=None):
    # The day book converges like every other citizen — by
    # identity, newest write wins per entry — with the one
    # harder law the store was founded under: a deletion
    # tombstones by identity, and a tombstoned entry is
    # honored over any copy no newer than the deletion (the
    # #253 discipline: a delete the operator watched happen
    # does not quietly undo itself from another seat's copy).
    raw = fetched["files"].get(SYNC_PREFIX + "daybook.json")
    if raw is None:
        return
    try:
        incoming = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return
    if not isinstance(incoming, dict):
        return
    inc_entries = []
    for raw_e in (incoming.get("entries") or []):
        e = _daybook_clean_entry(raw_e)
        if e is not None:
            inc_entries.append(e)
    day_tombs = {}
    if isinstance(tombs, dict):
        day_tombs = {str(k): float(v) for k, v in
                     (tombs.get("daybook") or {}).items()
                     if isinstance(v, (int, float))}
    changed = 0
    with _daybook_lock:
        store = _daybook_load()
        by_id = {e["id"]: e for e in store["entries"]}
        # Tombstones land first, over local copies no newer
        # than the deletion itself.
        kept = []
        for e in store["entries"]:
            at = day_tombs.get(e["id"])
            if at is not None and (e.get("updatedAt") or 0) <= at:
                changed += 1
                continue
            kept.append(e)
        store["entries"] = kept
        by_id = {e["id"]: e for e in store["entries"]}
        for e in inc_entries:
            at = day_tombs.get(e["id"])
            if at is not None and (e.get("updatedAt") or 0) <= at:
                continue  # the deletion is newer — it does not land
            cur = by_id.get(e["id"])
            if cur is None:
                store["entries"].append(e)
                by_id[e["id"]] = e
                changed += 1
            elif (e.get("updatedAt") or 0) > (cur.get("updatedAt") or 0):
                store["entries"] = [e if x["id"] == e["id"] else x
                                    for x in store["entries"]]
                by_id[e["id"]] = e
                changed += 1
        # One brief per day survives a merge: today's briefs
        # from two seats are two statements of one day — the
        # newer composition stands, the older steps aside.
        brief_days = {}
        for e in store["entries"]:
            if e["kind"] == "brief" and e["state"] == "standing":
                cur = brief_days.get(e.get("day") or "")
                if cur is None or (e.get("createdAt") or 0) > (cur.get("createdAt") or 0):
                    brief_days[e.get("day") or ""] = e
        for e in store["entries"]:
            if (e["kind"] == "brief" and e["state"] == "standing"
                    and brief_days.get(e.get("day") or "") is not e):
                e["state"] = "superseded"
                changed += 1
        inc_at = _daybook_num(incoming.get("lastBriefAt"))
        if inc_at > (store.get("lastBriefAt") or 0):
            store["lastBriefAt"] = inc_at
            store["lastBriefDay"] = str(incoming.get("lastBriefDay") or "")[:10]
        store["entries"] = store["entries"][-DAYBOOK_ENTRY_CAP:]
        _daybook_save(store)
        # A tombstone a surviving NEWER entry outlived is
        # spent: the copy won, the tombstone is dropped (the
        # standing law), so it stops travelling.
        if isinstance(tombs, dict) and isinstance(tombs.get("daybook"), dict):
            live = {e["id"]: (e.get("updatedAt") or 0)
                    for e in store["entries"]}
            for key in list(tombs["daybook"].keys()):
                if key in live and live[key] > tombs["daybook"][key]:
                    del tombs["daybook"][key]
    summary["daybookChanged"] = summary.get("daybookChanged", 0) + changed


# --- Web Push: the tray's phone door (v0.5.14e, #272) ----------------
# The #242 tray rings the operator's phone. Everything here
# is the node's own work: the VAPID keypair is minted once by
# this bridge and held in config/push.json (per-node, never
# synced); payloads are encrypted end-to-end for the
# subscription (RFC 8291) and signed (RFC 8292) with in-tree
# cryptography — P-256, ECDSA, ECDH, HKDF, AES-128-GCM,
# implemented below on the standard library alone and proved
# against the RFCs' published test vectors in the build's
# push1 bench (the RFC 8291 section 5 worked example
# reproduces byte for byte). The only parties a delivery ever
# touches are the push services the platform itself runs —
# the transport the Web Push standard is built on; there is
# no relay, no account, and no third-party service of ours.
# Parity is the law (Alex, 2026-11-17): whatever earns a
# tray record earns the phone copy, the payload is the tray
# record's own summary, the notification's tag is the tray
# record's id, an ack on any seat clears the phone copy, and
# the TTL is four hours. Delivery fires only from the seat
# that created the record; the outcome lands on the record's
# own push line in the tray, named, never silent.

def _b64u_dec(text):
    if isinstance(text, str):
        text = text.encode("ascii")
    return base64.urlsafe_b64decode(text + b"=" * (-len(text) % 4))


# --- P-256 (secp256r1) -------------------------------------------------
_P256_P = 0xFFFFFFFF00000001000000000000000000000000FFFFFFFFFFFFFFFFFFFFFFFF
_P256_B = 0x5AC635D8AA3A93E7B3EBBD55769886BC651D06B0CC53B0F63BCE3C3E27D2604B
_P256_G = (0x6B17D1F2E12C4247F8BCE6E563A440F277037D812DEB33A0F4A13945D898C296,
           0x4FE342E2FE1A7F9B8EE7EB4A7C0F9E162BCE33576B315ECECBB6406837BF51F5)
_P256_N = 0xFFFFFFFF00000000FFFFFFFFFFFFFFFFBCE6FAADA7179E84F3B9CAC2FC632551


def _ec_on_curve(pt):
    if pt is None:
        return False
    x, y = pt
    if not (0 <= x < _P256_P and 0 <= y < _P256_P):
        return False
    return (y * y - (x * x * x - 3 * x + _P256_B)) % _P256_P == 0


def _ec_add(p1, p2):
    if p1 is None:
        return p2
    if p2 is None:
        return p1
    x1, y1 = p1
    x2, y2 = p2
    if x1 == x2 and (y1 + y2) % _P256_P == 0:
        return None
    if p1 == p2:
        m = (3 * x1 * x1 - 3) * pow(2 * y1, -1, _P256_P) % _P256_P
    else:
        m = (y2 - y1) * pow(x2 - x1, -1, _P256_P) % _P256_P
    x3 = (m * m - x1 - x2) % _P256_P
    y3 = (m * (x1 - x3) - y1) % _P256_P
    return (x3, y3)


def _ec_mul(k, pt):
    # Scalar multiplication, double-and-add. k is reduced mod n;
    # the point at infinity is None.
    if pt is None or k % _P256_N == 0:
        return None
    k %= _P256_N
    out = None
    add = pt
    while k:
        if k & 1:
            out = _ec_add(out, add)
        add = _ec_add(add, add)
        k >>= 1
    return out


def _ec_pub_bytes(pt):
    return b"\x04" + pt[0].to_bytes(32, "big") + pt[1].to_bytes(32, "big")


def _ec_pub_parse(raw):
    # RFC 8291 section 7's receiving-end checks: the point must
    # be a well-formed uncompressed point ON the curve.
    if len(raw) != 65 or raw[0] != 0x04:
        return None
    pt = (int.from_bytes(raw[1:33], "big"), int.from_bytes(raw[33:65], "big"))
    return pt if _ec_on_curve(pt) else None


def _ec_keypair():
    priv = secrets.randbelow(_P256_N - 1) + 1
    return priv, _ec_mul(priv, _P256_G)


def _ecdsa_sign(priv, digest, k=None):
    # ES256 over a pre-computed SHA-256 digest; returns r||s.
    # The k parameter exists ONLY as a bench hook (RFC 6979
    # vectors name their k); production callers pass nothing.
    z = int.from_bytes(digest, "big")
    if k is None:
        k = secrets.randbelow(_P256_N - 1) + 1
    r = _ec_mul(k, _P256_G)[0] % _P256_N
    s = (pow(k, -1, _P256_N) * (z + r * priv)) % _P256_N
    return r.to_bytes(32, "big") + s.to_bytes(32, "big")


def _ecdsa_verify(pub_pt, digest, sig):
    if pub_pt is None or len(sig) != 64:
        return False
    r = int.from_bytes(sig[:32], "big")
    s = int.from_bytes(sig[32:], "big")
    if not (1 <= r < _P256_N and 1 <= s < _P256_N):
        return False
    z = int.from_bytes(digest, "big")
    w = pow(s, -1, _P256_N)
    pt = _ec_add(_ec_mul((z * w) % _P256_N, _P256_G),
                 _ec_mul((r * w) % _P256_N, pub_pt))
    return pt is not None and pt[0] % _P256_N == r


def _ecdh(priv, pub_pt):
    shared = _ec_mul(priv, pub_pt)
    if shared is None:
        return None
    return shared[0].to_bytes(32, "big")


# --- HKDF-SHA256 (RFC 5869, over hmac/hashlib) --------------------------
def _hkdf_extract(salt, ikm):
    return hmac.new(salt, ikm, hashlib.sha256).digest()


def _hkdf_expand(prk, info, length):
    out = b""
    t = b""
    i = 1
    while len(out) < length:
        t = hmac.new(prk, t + info + bytes([i]), hashlib.sha256).digest()
        out += t
        i += 1
    return out[:length]


# --- AES-128 (FIPS-197), encrypt direction only --------------------------
_AES_SBOX = bytes([
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16])
_AES_RCON = [0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1b, 0x36]


def _aes128_round_keys(key):
    # FIPS-197 key expansion: 44 words -> 11 round keys of 16.
    w = [list(key[i * 4:(i + 1) * 4]) for i in range(4)]
    for i in range(4, 44):
        t = list(w[i - 1])
        if i % 4 == 0:
            t = [_AES_SBOX[t[1]] ^ _AES_RCON[i // 4 - 1],
                 _AES_SBOX[t[2]], _AES_SBOX[t[3]], _AES_SBOX[t[0]]]
        w.append([w[i - 4][j] ^ t[j] for j in range(4)])
    return [bytes(sum(w[r * 4:(r + 1) * 4], [])) for r in range(11)]


def _aes_xtime(a):
    return ((a << 1) ^ 0x1b) & 0xff if a & 0x80 else (a << 1) & 0xff


def _aes128_encrypt_block(key, block):
    rks = _aes128_round_keys(key)
    s = [block[i] ^ rks[0][i] for i in range(16)]

    def shift_rows(x):
        # state is column-major: index = row + 4*col
        return [x[0], x[5], x[10], x[15], x[4], x[9], x[14], x[3],
                x[8], x[13], x[2], x[7], x[12], x[1], x[6], x[11]]

    for rnd in range(1, 10):
        s = [_AES_SBOX[b] for b in s]
        s = shift_rows(s)
        mixed = []
        for c in range(4):
            col = s[c * 4:(c + 1) * 4]
            mixed += [_aes_xtime(col[0]) ^ (_aes_xtime(col[1]) ^ col[1]) ^ col[2] ^ col[3],
                      col[0] ^ _aes_xtime(col[1]) ^ (_aes_xtime(col[2]) ^ col[2]) ^ col[3],
                      col[0] ^ col[1] ^ _aes_xtime(col[2]) ^ (_aes_xtime(col[3]) ^ col[3]),
                      (_aes_xtime(col[0]) ^ col[0]) ^ col[1] ^ col[2] ^ _aes_xtime(col[3])]
        s = [mixed[i] ^ rks[rnd][i] for i in range(16)]
    s = [_AES_SBOX[b] for b in s]
    s = shift_rows(s)
    return bytes(s[i] ^ rks[10][i] for i in range(16))


# --- GHASH + AES-128-GCM (SP 800-38D) ------------------------------------
def _ghash_mul(x_bytes, h_bytes):
    # GF(2^128) multiplication, SP 800-38D Algorithm 1 (the
    # bit-reflected convention: R = 11100001 || 0^120).
    z = 0
    v = int.from_bytes(h_bytes, "big")
    x = int.from_bytes(x_bytes, "big")
    for i in range(128):
        if (x >> (127 - i)) & 1:
            z ^= v
        if v & 1:
            v = (v >> 1) ^ 0xE1000000000000000000000000000000
        else:
            v >>= 1
    return z.to_bytes(16, "big")


def _ghash(h, data):
    # data length must already be a multiple of 16.
    y = b"\x00" * 16
    for i in range(0, len(data), 16):
        y = _ghash_mul(bytes(a ^ b for a, b in zip(y, data[i:i + 16])), h)
    return y


def _gcm_crypt(key, j0, data):
    # CTR mode from counter J0+1 (inc32 on the last 32 bits).
    out = bytearray()
    ctr = int.from_bytes(j0, "big")
    for i in range(0, len(data), 16):
        ctr = (ctr & ~0xFFFFFFFF) | ((ctr + 1) & 0xFFFFFFFF)
        ks = _aes128_encrypt_block(key, ctr.to_bytes(16, "big"))
        chunk = data[i:i + 16]
        out += bytes(a ^ b for a, b in zip(chunk, ks))
    return bytes(out)


def _gcm_tag(key, h, j0, ct, aad=b""):
    def pad16(x):
        return x + b"\x00" * (-len(x) % 16)
    lens = (len(aad) * 8).to_bytes(8, "big") + (len(ct) * 8).to_bytes(8, "big")
    s = _ghash(h, pad16(aad) + pad16(ct) + lens)
    ej0 = _aes128_encrypt_block(key, j0)
    return bytes(a ^ b for a, b in zip(s, ej0))


def _aes128gcm_encrypt(key, nonce12, plaintext):
    # 96-bit nonce only (RFC 8188's form). Returns ct||tag.
    h = _aes128_encrypt_block(key, b"\x00" * 16)
    j0 = nonce12 + b"\x00\x00\x00\x01"
    ct = _gcm_crypt(key, j0, plaintext)
    return ct + _gcm_tag(key, h, j0, ct)


def _aes128gcm_decrypt(key, nonce12, body):
    if len(body) < 16:
        return None
    ct, tag = body[:-16], body[-16:]
    h = _aes128_encrypt_block(key, b"\x00" * 16)
    j0 = nonce12 + b"\x00\x00\x00\x01"
    if not hmac.compare_digest(_gcm_tag(key, h, j0, ct), tag):
        return None
    return _gcm_crypt(key, j0, ct)


# --- RFC 8291 + RFC 8188: the aes128gcm content encoding ------------------
def _webpush_derive(auth_secret, ua_pub_raw, as_pub_raw, ecdh_secret, salt):
    # RFC 8291 section 3.3-3.4. Returns (cek, nonce, prk_key,
    # ikm, prk) — the intermediates ride along so the bench can
    # pin every one against the RFC's worked example.
    prk_key = _hkdf_extract(auth_secret, ecdh_secret)
    ikm = _hkdf_expand(prk_key, b"WebPush: info\x00" + ua_pub_raw + as_pub_raw, 32)
    prk = _hkdf_extract(salt, ikm)
    cek = _hkdf_expand(prk, b"Content-Encoding: aes128gcm\x00", 16)
    nonce = _hkdf_expand(prk, b"Content-Encoding: nonce\x00", 12)
    return cek, nonce, prk_key, ikm, prk


def _webpush_keys_sender(auth_secret, ua_pub_raw, as_priv, as_pub_raw, salt):
    ua_pt = _ec_pub_parse(ua_pub_raw)
    if ua_pt is None:
        return None
    ecdh_secret = _ecdh(as_priv, ua_pt)
    if ecdh_secret is None:
        return None
    return _webpush_derive(auth_secret, ua_pub_raw, as_pub_raw, ecdh_secret, salt)


def _webpush_keys_receiver(auth_secret, ua_pub_raw, ua_priv, as_pub_raw, salt):
    as_pt = _ec_pub_parse(as_pub_raw)
    if as_pt is None:
        return None
    ecdh_secret = _ecdh(ua_priv, as_pt)
    if ecdh_secret is None:
        return None
    return _webpush_derive(auth_secret, ua_pub_raw, as_pub_raw, ecdh_secret, salt)


def webpush_encrypt(ua_pub_b64, auth_b64, plaintext, as_priv=None, salt=None):
    # Encrypt one push payload for a subscription. The sender
    # (application server) keypair is ephemeral per RFC 8291;
    # as_priv/salt are parameters ONLY so the bench can replay
    # the RFC's worked example — production passes neither.
    ua_pub_raw = _b64u_dec(ua_pub_b64)
    auth_secret = _b64u_dec(auth_b64)
    if len(auth_secret) < 16 or _ec_pub_parse(ua_pub_raw) is None:
        return None
    if as_priv is None:
        as_priv, as_pt = _ec_keypair()
    else:
        as_pt = _ec_mul(as_priv, _P256_G)
    as_pub_raw = _ec_pub_bytes(as_pt)
    if salt is None:
        salt = secrets.token_bytes(16)
    keys = _webpush_keys_sender(auth_secret, ua_pub_raw, as_priv, as_pub_raw, salt)
    if keys is None:
        return None
    cek, nonce = keys[0], keys[1]
    rs = 4096
    if len(plaintext) + 1 + 16 > rs:
        return None  # one record only (RFC 8291 section 4)
    record = plaintext + b"\x02"
    body_ct = _aes128gcm_encrypt(cek, nonce, record)
    header = salt + rs.to_bytes(4, "big") + bytes([len(as_pub_raw)]) + as_pub_raw
    return header + body_ct


def webpush_decrypt(ua_priv, auth_b64, body):
    # The receiving end (the bench plays the user agent, holding
    # the subscription's private key): parse the RFC 8188
    # header, derive, decrypt, check the padding delimiter.
    auth_secret = _b64u_dec(auth_b64)
    if len(body) < 16 + 4 + 1 + 16:
        return None
    salt = body[:16]
    rs = int.from_bytes(body[16:20], "big")
    idlen = body[20]
    as_pub_raw = body[21:21 + idlen]
    ct = body[21 + idlen:]
    if len(as_pub_raw) != idlen or rs < 17 or not ct:
        return None
    ua_pt = _ec_mul(ua_priv, _P256_G)
    keys = _webpush_keys_receiver(auth_secret, _ec_pub_bytes(ua_pt), ua_priv, as_pub_raw, salt)
    if keys is None:
        return None
    return _webpush_decrypt_with(keys, ct, rs)


def _webpush_decrypt_with(keys, ct, rs):
    cek, base_nonce = keys[0], keys[1]
    out = b""
    seq = 0
    pos = 0
    while pos < len(ct):
        chunk = ct[pos:pos + rs]
        pos += len(chunk)
        nonce = bytes(a ^ b for a, b in zip(base_nonce, seq.to_bytes(12, "big")))
        rec = _aes128gcm_decrypt(cek, nonce, chunk)
        if rec is None:
            return None
        if not rec or rec[-1] not in (1, 2):
            return None
        out += rec[:-1].rstrip(b"\x00")
        seq += 1
    return out

def _b64u_dec_safe(text):
    try:
        return _b64u_dec(text)
    except Exception:
        return None


# --- The push store -------------------------------------------------
# config/push.json — THIS NODE's push state: its VAPID keypair
# and the subscriptions of the operator's own phones. It is
# per-node by design and never rides Sync (asserted in push1):
# a second node's subscriptions are its own, and the VAPID
# private key never leaves the node that minted it. The file
# is written 0600 — it holds key material.
PUSH_PATH = os.path.join(ROOT, "config", "push.json")
PUSH_TTL_SECONDS = 14400  # the ruled four hours
_push_lock = threading.Lock()


def _push_load():
    try:
        with open(PUSH_PATH) as f:
            data = json.load(f)
        if isinstance(data, dict):
            return data
    except Exception:
        pass
    return {}


def _push_save(store):
    try:
        os.makedirs(os.path.dirname(PUSH_PATH), exist_ok=True)
        tmp = PUSH_PATH + ".tmp"
        with open(tmp, "w") as f:
            json.dump(store, f, indent=2)
        os.chmod(tmp, 0o600)
        os.replace(tmp, PUSH_PATH)
        return True
    except Exception:
        return False


def _push_mint_vapid(store):
    # Minted ONCE, at first need, and kept: a node's VAPID
    # identity is stable across restarts (push1 asserts it),
    # because every subscription in the wild is bound to it.
    priv, pt = _ec_keypair()
    store["vapid"] = {"priv": _b64u(priv.to_bytes(32, "big")),
                      "pub": _b64u(_ec_pub_bytes(pt))}
    _push_save(store)
    return store["vapid"]


def push_vapid_public():
    with _push_lock:
        store = _push_load()
        vapid = store.get("vapid")
        if not (isinstance(vapid, dict) and vapid.get("priv") and vapid.get("pub")):
            vapid = _push_mint_vapid(store)
        return vapid["pub"]


def _push_vapid(store):
    vapid = store.get("vapid")
    if not (isinstance(vapid, dict) and vapid.get("priv") and vapid.get("pub")):
        vapid = _push_mint_vapid(store)
    return vapid


def _push_vapid_subject(cfg):
    # RFC 8292's sub: the node's own identity — its public
    # https name when DHD carries one, else a mailto on the
    # node's own hostname. Never another party's name.
    domain = ""
    try:
        domain = str(dhd_settings(cfg or load_config()).get("domain") or "")
    except Exception:
        domain = ""
    if domain:
        return "https://" + domain
    return "mailto:apex@" + (socket.gethostname() or "localhost")


def push_vapid_jwt(endpoint, cfg):
    # The VAPID authorization for one send (RFC 8292): an
    # ES256 JWT whose audience is the push service's own
    # origin, expiring within twelve hours, signed with this
    # node's VAPID private key. Returns (jwt, public key).
    vapid = _push_vapid(_push_load())
    priv_raw = _b64u_dec_safe(vapid["priv"]) or b""
    priv = int.from_bytes(priv_raw, "big")
    parts = urllib.parse.urlsplit(endpoint)
    claims = {"aud": parts.scheme + "://" + parts.netloc,
              "exp": int(time.time()) + 12 * 3600,
              "sub": _push_vapid_subject(cfg)}

    def enc(obj):
        return _b64u(json.dumps(obj, separators=(",", ":")).encode())

    signing = enc({"typ": "JWT", "alg": "ES256"}) + "." + enc(claims)
    sig = _ecdsa_sign(priv, hashlib.sha256(signing.encode()).digest())
    return signing + "." + _b64u(sig), vapid["pub"]


def _push_send_one(sub, payload_bytes, cfg):
    # One delivery attempt. Returns (status, error): the push
    # service's HTTP status, or (None, the failure named).
    body = webpush_encrypt(sub["keys"]["p256dh"], sub["keys"]["auth"],
                           payload_bytes)
    if body is None:
        return None, "the subscription's keys could not encrypt a payload"
    jwt, pub = push_vapid_jwt(sub["endpoint"], cfg)
    headers = {"Authorization": "vapid t=" + jwt + ", k=" + pub,
               "Content-Encoding": "aes128gcm",
               "Content-Type": "application/octet-stream",
               "TTL": str(PUSH_TTL_SECONDS)}
    req = urllib.request.Request(sub["endpoint"], data=body,
                                 headers=headers, method="POST")
    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            return resp.status, None
    except urllib.error.HTTPError as e:
        return e.code, None
    except Exception as e:
        return None, str(e)[:160]


def _push_note_death(sub, status):
    # A push service answering 404/410 has forgotten the
    # subscription: it comes off the list, and the death is
    # recorded where producer (h) of the bell can name it in
    # the tray — a phone that stops answering is never a
    # silent failure.
    with _push_lock:
        store = _push_load()
        subs = [s for s in (store.get("subscriptions") or [])
                if s.get("endpoint") != sub.get("endpoint")]
        store["subscriptions"] = subs
        dead = [d for d in (store.get("dead") or [])
                if d.get("id") != sub.get("id")]
        dead.append({"id": sub.get("id"), "label": sub.get("label"),
                     "host": urllib.parse.urlsplit(
                         str(sub.get("endpoint") or "")).hostname,
                     "status": status, "at": int(time.time() * 1000)})
        store["dead"] = dead[-8:]
        _push_save(store)


def push_deliver_record(rec, cfg):
    # The parity rule (Alex, 2026-11-17): whatever earns a
    # tray record earns the phone copy, carrying the tray
    # record's own summary — its title and body, verbatim —
    # tagged with the record's id so an ack can clear it.
    # Returns the outcome that lands on the record's push line.
    store = _push_load()
    subs = store.get("subscriptions") or []
    at = int(time.time() * 1000)
    if not subs:
        return {"state": "none", "detail": "no phone is subscribed",
                "at": at}
    payload = json.dumps(
        {"title": str(rec.get("title") or ""),
         "body": str(rec.get("body") or ""),
         "tag": str(rec.get("id") or ""),
         "url": "/#notice=" + str(rec.get("id") or "")},
        separators=(",", ":")).encode()
    sent = 0
    failures = []
    for sub in list(subs):
        status, err = _push_send_one(sub, payload, cfg)
        if status in (200, 201, 202):
            sent += 1
        elif status in (404, 410):
            _push_note_death(sub, status)
        else:
            host = urllib.parse.urlsplit(
                str(sub.get("endpoint") or "")).hostname or "the push service"
            failures.append((str(sub.get("label") or host)) + ": " +
                            (("HTTP " + str(status)) if status
                             else (err or "no answer")))
    if sent:
        detail = "pushed to %d phone%s" % (sent, "" if sent == 1 else "s")
        if failures:
            detail += "; " + "; ".join(failures)
        return {"state": "sent", "detail": detail, "at": at}
    return {"state": "failed",
            "detail": "; ".join(failures) or "no phone answered",
            "at": at}


def push_clear_record(rec_id, cfg):
    # The ack's other half: a payload carrying only the tag,
    # which the service worker reads as "close, show nothing".
    store = _push_load()
    subs = store.get("subscriptions") or []
    if not subs:
        return 0
    payload = json.dumps({"tag": str(rec_id), "clear": True},
                         separators=(",", ":")).encode()
    sent = 0
    for sub in list(subs):
        status, _err = _push_send_one(sub, payload, cfg)
        if status in (200, 201, 202):
            sent += 1
        elif status in (404, 410):
            _push_note_death(sub, status)
    return sent


def push_subscribe(data, cfg=None, device=""):
    # The subscribe door: validates the subscription the page
    # hands over before anything is stored — the endpoint must
    # be a push service (https; http only on loopback, where
    # the bench's stub door lives), the key a real P-256 point,
    # the auth secret at least 16 bytes. A re-subscribe of the
    # same endpoint replaces it; one carrying "replaces" (the
    # service worker's self-heal) inherits the old label and
    # clears that phone's death note. Returns (view, error).
    # v0.6.7e (#332): the record keys on the phone's durable
    # DOOR-DEVICE identity when the request carries one (the
    # handler resolves it from the session's device token):
    # a subscribe from a device that already holds a record
    # REPLACES that record — new endpoint, label and standing
    # carried — so one phone holds exactly one record, set
    # once, however often iOS re-mints its endpoint. Without
    # a device identity the endpoint remains the key, exactly
    # as before.
    data = data if isinstance(data, dict) else {}
    sub_in = data.get("subscription")
    if not isinstance(sub_in, dict):
        sub_in = data
    endpoint = str(sub_in.get("endpoint") or "")
    keys = sub_in.get("keys") if isinstance(sub_in.get("keys"), dict) else {}
    p256dh = str(keys.get("p256dh") or "")
    auth = str(keys.get("auth") or "")
    if not endpoint:
        return None, "The subscription carried no endpoint."
    parts = urllib.parse.urlsplit(endpoint)
    if parts.scheme == "https":
        pass
    elif parts.scheme == "http" and parts.hostname in ("127.0.0.1", "localhost", "::1"):
        pass
    else:
        return None, "The subscription's endpoint is not an https push service."
    if _ec_pub_parse(_b64u_dec_safe(p256dh) or b"") is None:
        return None, "The subscription's key is not a P-256 point."
    if len(_b64u_dec_safe(auth) or b"") < 16:
        return None, "The subscription's auth secret is too short."
    label = str(data.get("label") or "").strip()[:60]
    replaces = str(data.get("replaces") or "")
    device = str(device or "")[:80]
    with _push_lock:
        store = _push_load()
        subs = store.get("subscriptions") or []
        if not label:
            for s in subs:
                if s.get("endpoint") == replaces or \
                        (device and s.get("device") == device):
                    label = str(s.get("label") or "")
                    break
        subs = [s for s in subs
                if s.get("endpoint") != endpoint
                and s.get("endpoint") != replaces
                and not (device and s.get("device") == device)]
        rec = {"id": hashlib.sha256(endpoint.encode()).hexdigest()[:12],
               "endpoint": endpoint,
               "keys": {"p256dh": p256dh, "auth": auth},
               "label": label or ("Phone — " + (parts.hostname or "push service")),
               "device": device,
               "created": int(time.time() * 1000)}
        subs.append(rec)
        store["subscriptions"] = subs
        if rec["label"]:
            store["dead"] = [d for d in (store.get("dead") or [])
                             if d.get("label") != rec["label"]]
        _push_save(store)
    return {"id": rec["id"], "label": rec["label"],
            "device": rec["device"]}, None


def push_remove_record(record_id, cfg=None):
    # v0.6.7e (#332): any seat may remove a subscription
    # record — the roster-cleaning act the five-"iPhone" field
    # state needed. Removing a record stops that record's
    # ringing and nothing else: the tray is untouched, other
    # records stand, and the phone itself may subscribe again
    # whenever it likes. Returns the removal count.
    rid = str(record_id or "")
    if not rid:
        return {"ok": False, "error": "The removal named no record."}
    with _push_lock:
        store = _push_load()
        subs = store.get("subscriptions") or []
        kept = [s for s in subs if s.get("id") != rid]
        store["subscriptions"] = kept
        _push_save(store)
    return {"ok": True, "removed": len(subs) - len(kept)}


def push_unsubscribe(data, cfg=None):
    data = data if isinstance(data, dict) else {}
    endpoint = str(data.get("endpoint") or "")
    with _push_lock:
        store = _push_load()
        subs = store.get("subscriptions") or []
        kept = [s for s in subs if s.get("endpoint") != endpoint]
        store["subscriptions"] = kept
        _push_save(store)
    return {"ok": True, "removed": len(subs) - len(kept)}


def push_state_payload(cfg=None):
    store = _push_load()
    return {"ok": True,
            "vapidPublicKey": push_vapid_public(),
            "subscriptions": [
                {"id": s.get("id"), "label": s.get("label"),
                 "created": s.get("created"),
                 "device": s.get("device") or ""}
                for s in (store.get("subscriptions") or [])]}


def _notif_wanted_push_dead(cfg):
    # Producer (h): a subscribed phone the push service has
    # forgotten. The record stands until that phone subscribes
    # again (its death note clears at the subscribe door);
    # its Go target is Settings, where the push doors live.
    out = []
    try:
        store = _push_load()
    except Exception:
        return out
    for d in (store.get("dead") or []):
        rid = "push-dead:" + str(d.get("id") or "")
        out.append({
            "id": rid, "kind": "push-device",
            "title": "A phone stopped answering",
            "body": (str(d.get("label") or "A subscribed phone") +
                     " stopped answering push delivery (the service "
                     "answered " + str(d.get("status") or "gone") +
                     "). The tray still holds its records here, but "
                     "they no longer reach that phone — re-enable "
                     "phone notifications from that phone to heal "
                     "the door."),
            "target": {"view": "Settings"}, "count": None})
    return out


# The service worker, served at /sw.js. Embedded, not a tree
# file: the package's shape does not change to carry it. It
# holds no state — the tray on the bridge is the record; the
# worker only rings, clears, and opens the door.
PUSH_SERVICE_WORKER_JS = """
// Apex service worker (v0.5.14e, #272): the push doorbell's
// other half. Served by the bridge at /sw.js, scoped to the
// dashboard's own origin.
self.addEventListener("push", function (event) {
  var data = {};
  try { data = event.data ? event.data.json() : {}; } catch (e) { data = {}; }
  if (data.clear) {
    // An ack on any seat clears the phone copy: close every
    // notification carrying the tray record's tag, show none.
    event.waitUntil(registration.getNotifications({ tag: data.tag }).then(function (ns) {
      ns.forEach(function (n) { n.close(); });
    }));
    return;
  }
  var opts = { body: data.body || "", tag: data.tag || undefined,
               data: { url: data.url || "/" } };
  event.waitUntil(registration.showNotification(data.title || "Navi — needs your eyes", opts));
});
self.addEventListener("notificationclick", function (event) {
  event.notification.close();
  var url = (event.notification.data && event.notification.data.url) || "/";
  event.waitUntil(clients.matchAll({ type: "window", includeUncontrolled: true }).then(function (wins) {
    for (var i = 0; i < wins.length; i++) {
      if ("focus" in wins[i]) { wins[i].navigate(url); return wins[i].focus(); }
    }
    return clients.openWindow(url);
  }));
});
self.addEventListener("pushsubscriptionchange", function (event) {
  // The platform rotated the subscription: re-subscribe and
  // hand the bridge the new endpoint, naming the old one so
  // the bridge carries the label over — the door heals
  // itself instead of dying silently.
  var oldSub = event.oldSubscription;
  event.waitUntil((async function () {
    try {
      var key = oldSub && oldSub.options ? oldSub.options.applicationServerKey : undefined;
      var sub = await registration.pushManager.subscribe(
        { userVisibleOnly: true, applicationServerKey: key });
      await fetch("/api/push/subscribe", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ subscription: sub.toJSON(),
                               replaces: oldSub ? oldSub.endpoint : "" })
      });
    } catch (e) { /* the death, if any, is named in the tray */ }
  })());
});
"""



# --- Notifications (v0.4.7e, #242; ruling Alex 2026-11-16) -----------
# Navi announces notifications in chat herself — the bell/panel is
# the persistent record, her word is the tap on the shoulder, and
# the announcement is generated FROM the bridge's record, never a
# substitute for it. If a record is cleared/retired, she does not
# still mention it: retirement below deletes the record outright,
# and a record is announced exactly once, when it is born (the
# announced flag is the receipt). This is rules 14 + 15 at work —
# she senses what is waiting and says so in the room, in the same
# pass that records it.
#
# The store is bridge-held, one plain file (config/notifications.json,
# the chat-routes store pattern: a module path constant, a lock, a
# load that cleans, a store function; the path is read from the
# constant at call time). QUIET DISCIPLINE: only acts and decisions
# notify — a knock at a gateway, a device waiting at the door,
# case entries waiting for review, an update staged. Routine syncs,
# successful borrows and feed fetches NEVER notify. One sync
# exception, ruled in v0.4.10e (#254): a sync that materially
# changed things — a fold, a pairing repair, a tombstone honored —
# is an act, and producer (e) below speaks for exactly those.
# A second ruled addition, v0.5.4e (#267): FediChat arrivals —
# lines that LAND in a room from other members are an act the
# operator waits on without knowing it, so producer (f) speaks
# for the bridge's arrival journal: one record per arrival
# round, naming the room. The operator's own lines never
# journal, so they never notify.
#
# A record: {id, kind, title, body, target, created, read,
# announced, count}. Reconcile computes the WANTED set from the
# producers and folds it into the store: a wanted id with no
# record is born (and announced); a record still wanted is
# refreshed in place — and if its count grew, it goes unread
# again, because new material needs eyes; a record no longer
# wanted is retired (the knock was answered, the review emptied,
# the update applied). Each producer is wrapped on its own: one
# producer's failure yields no items from it and retires NONE of
# its kind's records — a transient error must never read as
# "nothing is waiting anymore".

NOTIF_PATH = os.path.join(ROOT, "config", "notifications.json")
NOTIF_CAP = 64
NOTIF_PAYLOAD_CAP = 20
NOTIF_RECONCILE_SECONDS = 10
_notif_lock = threading.Lock()
_NOTIF_STATE = {"at": 0.0}


def _notif_clean(raw):
    # One record, cleaned — the load's discipline (the chat-routes
    # cleaning idiom): a malformed row is dropped, never repaired
    # into a half-truth the bell would then vouch for.
    if not isinstance(raw, dict):
        return None
    rid = str(raw.get("id") or "").strip()[:200]
    if not rid:
        return None
    kind = str(raw.get("kind") or "").strip()[:40]
    if not kind:
        return None
    target = raw.get("target") if isinstance(raw.get("target"), dict) else {}
    created = raw.get("created")
    if not isinstance(created, (int, float)) or isinstance(created, bool):
        created = 0
    count = raw.get("count")
    if not isinstance(count, int) or isinstance(count, bool):
        count = None
    push = raw.get("push") if isinstance(raw.get("push"), dict) else None
    # v0.6.3e (#319): the plain factual record rides beside the
    # voiced summary (the record's title/body since this build).
    # A record stored by an older build has no fact — its own
    # title/body ARE the fact, and the next reconcile voices it.
    fact = raw.get("fact") if isinstance(raw.get("fact"), dict) else None
    if fact is not None:
        fact = {"title": str(fact.get("title") or "")[:200],
                "body": str(fact.get("body") or "")[:600]}
    else:
        fact = {"title": str(raw.get("title") or "")[:200],
                "body": str(raw.get("body") or "")[:600]}
    return {
        "id": rid,
        "kind": kind,
        "title": str(raw.get("title") or "")[:200],
        "body": str(raw.get("body") or "")[:600],
        "fact": fact,
        "target": dict(target),
        "created": created,
        "read": raw.get("read") is True,
        "announced": raw.get("announced") is True,
        "count": count,
        "push": push,
    }


def _notif_load():
    try:
        with open(NOTIF_PATH) as f:
            data = json.load(f)
        rows = data.get("records") if isinstance(data, dict) else None
        if isinstance(rows, list):
            out = []
            seen = set()
            for row in rows:
                rec = _notif_clean(row)
                if rec is not None and rec["id"] not in seen:
                    seen.add(rec["id"])
                    out.append(rec)
            return out[-NOTIF_CAP:]
    except Exception:
        pass
    return []


def _notif_save(records):
    try:
        os.makedirs(os.path.dirname(NOTIF_PATH), exist_ok=True)
        tmp = NOTIF_PATH + ".tmp"
        with open(tmp, "w") as f:
            json.dump({"records": records}, f, indent=2)
        os.replace(tmp, NOTIF_PATH)
        return True
    except Exception:
        return False


def _notif_wanted_fed_knocks(cfg):
    # Producer (a): a knock at a gateway this node keeps — each
    # pending join on a GATEWAY fedinet (member fedinets hold no
    # knock queue here; the gateway answers knocks). The pending
    # entry carries nodeId + the knock's card; the node's name
    # rides the card, exactly as federation_payload reads it.
    items = []
    for fed in federation_settings(cfg):
        if fed.get("role") != "gateway":
            continue
        fed_id = str(fed.get("id") or "")
        fed_name = str(fed.get("name") or fed_id)
        if not fed_id:
            continue
        for pending in (fed.get("pendingJoins") or []):
            if not isinstance(pending, dict):
                continue
            node_id = str(pending.get("nodeId") or "")
            if not node_id:
                continue
            card = pending.get("card") if isinstance(pending.get("card"), dict) else {}
            node_name = str(card.get("name") or "") or "A node"
            items.append({
                "id": "fed-knock:" + fed_id + ":" + node_id,
                "kind": "fed-knock",
                "title": "A knock at " + fed_name,
                "body": (node_name + " (" + node_id[:12]
                         + ") is asking to join."),
                "target": {"view": "Federation", "fedinetId": fed_id,
                           "nodeId": node_id},
                "count": 1,
            })
    return items


def _notif_wanted_door_devices(cfg):
    # Producer (b): a device asking to pair at the door. Pairwise
    # Mesh pairing is card-consent with no queue in this product;
    # the pending-answer queue that exists is the door's device
    # approvals (door_devices_payload) — that is this producer.
    items = []
    for dev in door_devices_payload(cfg):
        if not isinstance(dev, dict) or dev.get("status") != "pending":
            continue
        dev_id = str(dev.get("id") or "")
        if not dev_id:
            continue
        name = str(dev.get("name") or "Unnamed device")
        items.append({
            "id": "door-device:" + dev_id,
            "kind": "door-device",
            "title": "A device is asking to pair",
            "body": (name + " is waiting for your word — approve it "
                     "or turn it away at the Door (Settings, or the "
                     "DHD page — the same card stands at both)."),
            "target": {"view": "DHD"},
            "count": 1,
        })
    return items


def _notif_wanted_case_reviews():
    # Producer (c): case entries waiting for the operator's
    # review — each case head's unverified count (leads Navi
    # filed, awaiting a verdict) above zero.
    items = []
    for head in (cases_list_payload().get("cases") or []):
        if not isinstance(head, dict):
            continue
        n = head.get("unverified")
        if not isinstance(n, int) or isinstance(n, bool) or n <= 0:
            continue
        slug = str(head.get("slug") or "")
        if not slug:
            continue
        items.append({
            "id": "case-review:" + slug,
            "kind": "case-review",
            "title": str(head.get("name") or slug),
            "body": (str(n) + " unverified entries are waiting "
                     "for your review"),
            "target": {"view": "CaseFiles", "case": slug},
            "count": n,
        })
    return items


def _notif_wanted_update():
    # Producer (d): an update staged and waiting on the Updater —
    # the background checker's cached verdict, read as-is.
    payload = update_bg_payload()
    if not isinstance(payload, dict) or not payload.get("updateAvailable"):
        return []
    latest = str(payload.get("latest") or "")
    if not latest:
        return []
    return [{
        "id": "update:" + latest,
        "kind": "update",
        "title": "Apex " + latest + " is ready",
        "body": "An Edge update is staged and waiting on the Updater.",
        "target": {"view": "Settings"},
        "count": 1,
    }]


def _notif_wanted_sync():
    # Producer (e) (v0.4.10e, #254): a sync that MATERIALLY
    # changed things — ghost threads folded into the main chat
    # (#251), a case/chat pairing repaired or a gap folded away
    # (#252), another seat's tombstone honored here (#253) — is
    # an act, not routine traffic, and it is announced from the
    # bridge's record like everything else here. The wanted set
    # holds only the latest material receipt, so the record
    # retires when a newer material round supersedes it; a
    # routine sync produces nothing, exactly as the quiet
    # discipline above demands.
    state = _sync_state_load()
    receipt = state.get("lastReceipt") or {}
    if not receipt.get("material"):
        return []
    counts = receipt.get("counts") or {}
    parts = []
    if counts.get("foldedGhosts"):
        n = counts["foldedGhosts"]
        parts.append(str(n) + " ghost thread" + ("s" if n != 1 else "")
                     + " folded into the main chat ("
                     + str(counts.get("foldedMessages") or 0) + " messages carried in)")
    if counts.get("mergedMessages"):
        n = counts["mergedMessages"]
        parts.append(str(n) + " message" + ("s" if n != 1 else "")
                     + " merged into the main thread")
    if counts.get("pairsRepaired"):
        n = counts["pairsRepaired"]
        parts.append(str(n) + " case/chat pair" + ("s" if n != 1 else "") + " made whole")
    for t in (counts.get("gapTitles") or [])[:3]:
        parts.append("the chat of “" + str(t) + "” folded away — its case never crossed")
    if counts.get("tombstones"):
        n = counts["tombstones"]
        parts.append("a deletion from your other machine was honored ("
                     + str(n) + " thread" + ("s" if n != 1 else "") + " removed here)")
    if not parts:
        return []
    return [{
        "id": "sync:" + str(receipt.get("roundId") or receipt.get("at") or ""),
        "kind": "sync",
        "title": "Sync changed things here",
        "body": "; ".join(parts) + ".",
        "target": {"view": "Chat"},
        "count": 1,
    }]


def _notif_wanted_purge():
    # Producer (v0.6.4e, #323): the account gate's purge is an
    # act done in the operator's name on their own machine —
    # it is announced like every other act, in Navi's voice
    # (kind "sync" rides the #319 sync voice), from the
    # standing purge record. The record retires when a newer
    # purge supersedes it; no purge, no notice.
    state = _sync_state_load()
    p = state.get("lastPurge") or {}
    if not p.get("at") or not p.get("fact"):
        return []
    return [{
        "id": "purge:" + str(p.get("at") or ""),
        "kind": "sync",
        "title": "Sync removed another account's material",
        "body": str(p.get("fact") or ""),
        "target": {"view": "Chat"},
        "count": 1,
    }]


def _notif_wanted_fedichat(cfg):
    # Producer (f) (v0.5.4e, #267): FediChat arrivals. The
    # bridge journals one entry per arrival round at merge
    # time (_fedichat_journal_arrival); this producer speaks
    # for the journal — one record per round, naming the
    # room, its count the round's line count. A single-line
    # round names its author (operator name where the room
    # holds one, the journaled node name otherwise) and
    # carries the line itself. Rounds retire as the journal's
    # cap ages them out; read state is the tray's own.
    items = []
    rounds = _fedichat_arrivals_load()
    if not rounds:
        return items
    feds = {str(f.get("id") or ""): f for f in federation_settings(cfg)}
    op_cache = {}
    for r in rounds:
        fed_id = str(r.get("fed") or "")
        room = str(r.get("room") or "#fedichat")
        count = r.get("count") if isinstance(r.get("count"), int) else 1
        authors = [str(a or "") for a in (r.get("authors") or [])]
        names = [str(n or "") for n in (r.get("names") or [])]
        if fed_id not in op_cache:
            fed = feds.get(fed_id)
            op_cache[fed_id] = _fedichat_op_names(
                fedichat_log_read(fed_id)) if fed is not None else {}
        op_names = op_cache[fed_id]

        def display(i):
            if i < len(authors) and op_names.get(authors[i]):
                return op_names[authors[i]]
            if i < len(names) and names[i]:
                return names[i]
            return "A member"
        if count == 1:
            title = "New line in " + room
            body = display(0) + ": " + str(r.get("firstBody") or "")
        else:
            title = "New lines in " + room
            seen = []
            for i in range(len(authors)):
                d = display(i)
                if d not in seen:
                    seen.append(d)
            body = ("From " + ", ".join(seen[:3])
                    + (" and others" if len(seen) > 3 else "") + ".")
        items.append({
            "id": str(r.get("id") or ""),
            "kind": "fedichat",
            "title": title,
            "body": body,
            "target": {"view": "FediChat", "fedinetId": fed_id,
                       "roomId": str(r.get("roomId") or "")},
            "count": count,
        })
    return items


# -- Her voice on the alert path (v0.6.3e, #319) -------------------
# Ruled on the #272 field proof: the lock screen read "Reminder
# / from Apex / this is a test" — the machinery announcing
# itself, in the product's name. Alex: "It should be Navi
# sending the alerts. Not Apex. With Navi's own flare to them."
# So the tray record's SUMMARY — the title and body the bell
# shows and the push payload mirrors verbatim (#272 parity,
# unchanged) — is composed in her voice: her name leads the
# title, and a written flare line per producer class wraps the
# fact. The fact itself is never edited: reminder text, times,
# names and counts ride inside the body exactly as the producer
# stated them, and the plain factual record stays on the tray
# record's own "fact" — the detail keeps the structured truth,
# the summary is the voice. The banks are written lines, chosen
# deterministically (the #288/#304 pattern — the record's class
# and count draw the line; the same draw always lands the same
# line, consecutive counts never repeat it) — NO model call on
# the alert path. Borrowed and federation-served alerts are
# announced by the LOCAL Navi: composition happens here, at
# record birth on this seat, whoever's machinery woke her.

NAVI_VOICE_TITLES = {
    "reminder": "Navi — Reminder",
    "order": "Navi — Standing order",
    "pulse": "Navi — Pulse",
    "fedichat": "Navi — FediChat",
    "sync": "Navi — Sync",
    "update": "Navi — Update",
    "fed-knock": "Navi — Federation",
    "door-device": "Navi — At the door",
    "case-review": "Navi — Case Files",
    "push-device": "Navi — Phone door",
}

NAVI_VOICE_LINES = {
    "reminder": ["On your feet — ",
                 "Time's up — ",
                 "Ring ring — "],
    "order": ["Done while you weren't looking — ",
              "Your standing order, handled — ",
              "Ran it, as ordered — "],
    "pulse": ["Something moved — ",
              "Fresh off the wire — ",
              "Your watch paid off — "],
    "fedichat": ["The room's talking — ",
                 "Word from the room — ",
                 "Someone's at the table — "],
    "sync": ["I tidied the room — ",
             "Across your machines — ",
             "Housekeeping, done — "],
    "update": ["A present for you — ",
               "Fresh iron on the bench — ",
               "Something new, staged and waiting — "],
    "fed-knock": ["Knock knock — ",
                  "Someone at the gate — ",
                  "A knock worth hearing — "],
    "door-device": ["At the door — ",
                    "Company — ",
                    "Someone's waiting — "],
    "case-review": ["For your eyes — ",
                    "The case file stirs — ",
                    "Homework, of a kind — "],
    "push-device": ["A door went quiet — ",
                    "One phone's gone dark — ",
                    "Door trouble — "],
    "_fallback": ["A word — ",
                  "Heads up — ",
                  "From my desk — "],
}


def notif_compose_summary(kind, fact_title, fact_body, count=None,
                          seed=""):
    # One record's summary in her voice: (title, body). The
    # title leads with her name; the body is one flare line
    # from the class's bank wrapped around the fact, verbatim —
    # the fact is the producer's own body text, untouched. The
    # draw is a stable hash of the class and the record's seed,
    # walked by the count: the same class + count always draws
    # the same line, and consecutive counts draw different
    # lines (the bank walks, never repeats a neighbor).
    kind = str(kind or "")
    title = NAVI_VOICE_TITLES.get(kind, "Navi")
    lines = NAVI_VOICE_LINES.get(kind) or NAVI_VOICE_LINES["_fallback"]
    n = count if isinstance(count, int) and not isinstance(count, bool) \
        and count > 0 else 1
    draw = int(hashlib.sha256(
        (kind + "|" + str(seed)).encode("utf-8")).hexdigest()[:8], 16)
    fact_body = str(fact_body or "")
    # The fact rides whole: where the producer's title carries
    # substance its body lacks (a room's name, a case's name, a
    # version), the title leads the body — verbatim, as stated —
    # so the summary never knows less than the fact did. Pure
    # labels ("Reminder", "A standing order ran") add nothing
    # and stay in the detail.
    generic = {"Reminder", "A standing order ran",
               "Sync changed things here",
               "A device is asking to pair",
               "A phone stopped answering"}
    lead = ""
    ft = str(fact_title or "")
    if ft and ft not in generic and ft not in fact_body:
        lead = ft + " — "
    body = lines[(draw + n) % len(lines)] + lead + fact_body
    return title[:200], body[:600]


def _notif_announce_line(rec):
    # Navi's line, composed FROM THE RECORD (deterministic
    # templates per kind — no model call): her voice, warm and
    # factual, naming the act that waits. The substance is the
    # record's FACT (v0.6.3e, #319: the summary is already her
    # voice; the announcement frames the plain fact, so the
    # room never hears the flare twice).
    framing = {
        "fed-knock": "Knock knock — ",
        "door-device": "At the door — ",
        "case-review": "Case Files — ",
        "update": "Good news — ",
        "sync": "Sync — ",
        "fedichat": "FediChat — ",
        "pulse": "Pulse — ",
    }.get(rec.get("kind"), "A word — ")
    fact = rec.get("fact") if isinstance(rec.get("fact"), dict) else {}
    line = framing + str(fact.get("title") or rec.get("title") or "") \
        + ": " + str(fact.get("body") or rec.get("body") or "")
    if rec.get("kind") == "fed-knock":
        line += " It waits on your word in Federation."
    return line


def _notif_announce(rec):
    # The tap on the shoulder: Navi's line lands in the MAIN chat
    # as her own message, carrying the record's id in "notice" so
    # the room and the record point at each other. A chat-store
    # failure returns False and breaks nothing — the caller
    # leaves announced=False and a later pass retries (once per
    # pass, never a loop inside one).
    try:
        line = _notif_announce_line(rec)
        now_ms = int(time.time() * 1000)
        chat = None
        try:
            with open(_chat_path(MAIN_CHAT_ID), encoding="utf-8") as f:
                chat = json.load(f)
        except Exception:
            chat = None
        if not isinstance(chat, dict):
            chat = {"id": MAIN_CHAT_ID, "title": "Chat", "messages": [],
                    "createdAt": now_ms, "updatedAt": now_ms}
        chat["id"] = MAIN_CHAT_ID
        if not chat.get("title"):
            chat["title"] = "Chat"
        if not isinstance(chat.get("createdAt"), (int, float)):
            chat["createdAt"] = now_ms
        msgs = chat.get("messages")
        if not isinstance(msgs, list):
            msgs = []
        # Cross-seat convergence (v0.5.13e, #287): a pulse
        # record's id is a fact of the batch itself, and chat
        # rides Sync — the same batch may already stand
        # announced in this room under the other seat's
        # store-local record id. Never announce one batch
        # twice: the batch digest is the identity.
        if rec.get("kind") == "pulse":
            _bd = str(rec.get("id") or "").rsplit(":", 1)[-1]
            if _bd and any(
                    isinstance(x, dict)
                    and str(x.get("notice") or "").startswith("pulse:")
                    and str(x.get("notice") or "").rsplit(":", 1)[-1] == _bd
                    for x in msgs):
                return True
        msgs.append({"who": "NAVI", "text": line, "exec": None,
                     "notice": rec["id"], "ts": now_ms})
        chat["messages"] = msgs
        chat["updatedAt"] = now_ms
        return chat_write(chat) is not None
    except Exception:
        return False


def _notif_payload_from_records(records):
    items = [{
        "id": r["id"], "kind": r["kind"], "title": r["title"],
        "body": r["body"], "target": r["target"],
        "created": r["created"], "read": r["read"], "count": r["count"],
        "push": r.get("push"),
        # v0.6.3e (#319): the plain factual record, for the
        # panel's detail — the summary above is the voice.
        "fact": r.get("fact"),
    } for r in records]
    # Unread first, then newest first — the bell's order.
    items.sort(key=lambda it: (it["read"], -(it["created"] or 0)))
    return {
        "unread": sum(1 for r in records if not r["read"]),
        "items": items[:NOTIF_PAYLOAD_CAP],
    }


def notifications_reconcile(cfg=None):
    # One pass: wanted set in, store folded, births announced,
    # retirements deleted, save only if anything changed. The
    # return is the payload (see notifications_payload).
    if cfg is None:
        cfg = load_config()
    born = []
    with _notif_lock:
        records = _notif_load()
        # v0.5.14e (#272): what the main chat already carries
        # announced, captured BEFORE this pass announces
        # anything — a birth whose announcement already stands
        # here was written by another seat and converged; it is
        # a convergence, not a birth, and never re-pushes.
        pre_announced = set()
        try:
            for c in chats_all():
                if str(c.get("id")) == str(MAIN_CHAT_ID):
                    for m in c.get("messages") or []:
                        if isinstance(m, dict) and m.get("notice"):
                            pre_announced.add(str(m["notice"]))
                    break
        except Exception:
            pre_announced = set()
        wanted = {}
        ran_kinds = set()
        producers = (
            ("fed-knock", lambda: _notif_wanted_fed_knocks(cfg)),
            ("door-device", lambda: _notif_wanted_door_devices(cfg)),
            ("case-review", lambda: _notif_wanted_case_reviews()),
            ("update", lambda: _notif_wanted_update()),
            ("sync", lambda: _notif_wanted_sync()),
            ("sync-purge", lambda: _notif_wanted_purge()),
            ("fedichat", lambda: _notif_wanted_fedichat(cfg)),
            ("pulse", lambda: _notif_wanted_pulse(cfg)),
            ("seraph", lambda: _notif_wanted_seraph(cfg)),
            ("reminder", lambda: _notif_wanted_day(cfg)),
            ("order", lambda: _notif_wanted_day(cfg)),
            ("push-dead", lambda: _notif_wanted_push_dead(cfg)),
        )
        for kind, produce in producers:
            try:
                items = produce()
            except Exception:
                # A failed producer yields no items and retires
                # none of its kind's records (see the banner).
                continue
            ran_kinds.add(kind)
            for item in (items or []):
                if isinstance(item, dict) and item.get("id"):
                    wanted[str(item["id"])] = item
        changed = False
        by_id = {r["id"]: r for r in records}
        now = time.time()
        for wid, item in wanted.items():
            rec = by_id.get(wid)
            # v0.6.3e (#319): the producer's item is the FACT;
            # the record's title/body are the summary in Navi's
            # voice, composed from it at birth and re-composed
            # whenever the fact moves (a count climbing draws
            # the class bank's next line). The push payload
            # mirrors the summary verbatim, as it always has.
            fact = {"title": str(item.get("title") or "")[:200],
                    "body": str(item.get("body") or "")[:600]}
            item_count = (item.get("count")
                          if isinstance(item.get("count"), int)
                          and not isinstance(item.get("count"), bool)
                          else None)
            title, body = notif_compose_summary(
                str(item.get("kind") or ""), fact["title"], fact["body"],
                count=item_count, seed=wid)
            if rec is None:
                rec = {
                    "id": wid,
                    "kind": str(item.get("kind") or ""),
                    "title": title,
                    "body": body,
                    "fact": fact,
                    "target": (dict(item["target"])
                               if isinstance(item.get("target"), dict) else {}),
                    "created": now,
                    "read": False,
                    "announced": False,
                    "count": item_count,
                }
                records.append(rec)
                by_id[wid] = rec
                born.append(rec)
                changed = True
            else:
                target = (dict(item["target"])
                          if isinstance(item.get("target"), dict) else {})
                if (rec.get("fact") != fact or rec["title"] != title
                        or rec["body"] != body
                        or rec["target"] != target):
                    rec["fact"] = fact
                    rec["title"], rec["body"], rec["target"] = title, body, target
                    changed = True
                new_count = item.get("count")
                if isinstance(new_count, int) and not isinstance(new_count, bool):
                    if isinstance(rec.get("count"), int) \
                            and new_count > rec["count"] and rec["read"]:
                        # New material needs eyes again.
                        rec["read"] = False
                        changed = True
                    if rec.get("count") != new_count:
                        rec["count"] = new_count
                        changed = True
            if not rec["announced"]:
                if _notif_announce(rec):
                    rec["announced"] = True
                    changed = True
        kept = []
        for rec in records:
            if rec["id"] in wanted or rec["kind"] not in ran_kinds:
                kept.append(rec)
            else:
                # Retired: the act was answered, the queue
                # emptied, the update applied — the record goes,
                # and with it any reason to mention it again.
                changed = True
        records = kept
        if len(records) > NOTIF_CAP:
            records = records[-NOTIF_CAP:]
            changed = True
        if changed:
            _notif_save(records)
        _NOTIF_STATE["at"] = time.time()
        payload = _notif_payload_from_records(records)
    # v0.5.14e (#272): the phone door rings for the records
    # BORN in this pass — out here, never under the bell's lock
    # (network I/O does not hold it), and only from the seat
    # that created the record. The outcome lands on the
    # record's own push line, where the panel shows it; a push
    # failure is named there, never silent, and never breaks
    # the reconcile that carries it.
    for rec in born:
        rid = str(rec.get("id") or "")
        converged = rid in pre_announced or (
            rec.get("kind") == "pulse" and any(
                a.startswith("pulse:")
                and a.rsplit(":", 1)[-1] == rid.rsplit(":", 1)[-1]
                and a != rid for a in pre_announced))
        if converged:
            outcome = {"state": "none",
                       "detail": "born on another seat; not re-pushed",
                       "at": int(time.time() * 1000)}
        else:
            try:
                outcome = push_deliver_record(rec, cfg)
            except Exception as e:
                outcome = {"state": "failed",
                           "detail": "push machinery error: " + str(e)[:120],
                           "at": int(time.time() * 1000)}
        with _notif_lock:
            current = _notif_load()
            for live in current:
                if live["id"] == rid:
                    live["push"] = outcome
                    _notif_save(current)
                    payload = _notif_payload_from_records(current)
                    break
    return payload


def notifications_payload(cfg=None, force=False):
    # The bell's read: a throttled reconcile (at most once per
    # NOTIF_RECONCILE_SECONDS unless forced) over the store, so
    # /api/status can carry it on every poll without re-running
    # the producers each time. Between reconciles the answer is
    # read straight from the store as it stands.
    if not force and _NOTIF_STATE["at"] \
            and (time.time() - _NOTIF_STATE["at"] < NOTIF_RECONCILE_SECONDS):
        return _notif_payload_from_records(_notif_load())
    return notifications_reconcile(cfg)


def notifications_mark_read(ids=None, mark_all=False, cfg=None):
    # The bell's write: mark the named records read (or all of
    # them). Unknown ids are ignored, not errors — the panel may
    # hold a record a reconcile has already retired. The answer
    # is the fresh payload.
    want = None
    if isinstance(ids, (list, tuple, set)):
        want = {str(i) for i in ids}
    cleared = []
    with _notif_lock:
        records = _notif_load()
        changed = False
        for rec in records:
            if (mark_all or (want is not None and rec["id"] in want)) \
                    and not rec["read"]:
                rec["read"] = True
                changed = True
                cleared.append(rec["id"])
        if changed:
            _notif_save(records)
    # v0.5.14e (#272): an ack on ANY seat clears the phone
    # copy — for each record that just went read, a clearing
    # push carrying only its tag tells every subscribed phone
    # to close the notification. Best-effort by design: the
    # tray is the record; the phone copy is a doorbell.
    for rid in cleared:
        try:
            push_clear_record(rid, cfg)
        except Exception:
            pass
    return notifications_payload(cfg)


# -- The Row Feed (v0.4.9e, #248) --------------------------------------
# Context Awareness: Apex and nocturnerow.com are one
# ecosystem, and the bridge reads the Row's half of it —
# Dispatches, Roadmap chapters, her page and the Apex page,
# release signals, the gate's seven steps — through
# rowfeed.py, caching the record at config/row-feed.json.
# Refresh discipline mirrors the bell's: throttled, and the
# cache always stands — a failed fetch keeps the last good
# sections standing and its error is NAMED in the record; no
# cache and no fetch, and the context blocks below say she
# cannot reach the Row right now. No silent empties, ever.
ROWFEED_PATH = os.path.join(ROOT, "config", "row-feed.json")
ROWFEED_TTL_SECONDS = 900
_ROWFEED_STATE = {"at": 0.0, "refreshing": False}
_rowfeed_lock = threading.Lock()

ROW_TEACHING = (
    "The Row: Apex and nocturnerow.com are one ecosystem. The site carries "
    "your Dispatches, your Roadmap, your own page, and your release signals, "
    "and the bridge fetches that public record and shows it to you in "
    "[The Row] blocks, each naming when it was fetched. When the operator "
    "asks for news from the Row, updates from the crew, or anything about "
    "the Dispatches, answer from the supplied Dispatch: summarize it in "
    "your own words and name its date. When they ask what is new with you "
    "— your latest update, your version — answer from the release signals "
    "and your running build's facts, as supplied. When they ask what you "
    "can do, answer from your documentation index, as supplied — the "
    "shipped corpus, page by page. Grounding is absolute: speak only from "
    "what the blocks show; if the record could not be reached, or a fact "
    "is not in it, say exactly that — never invent a Dispatch, a release, "
    "a date, or a capability, and name a figure's age when it is stale. "
    "Your pace: when a [Your pace] block rides, your measured generation "
    "on this hardware is slow across several generations — you may say "
    "you are sluggish here, plainly, as the hardware's fact. The remedy "
    "is the road: borrow through the Mesh, or join the federation. You "
    "may offer to walk the operator through joining, using the gate's "
    "seven steps when they ride in your context — an offer you make "
    "once per conversation, never an act you perform yourself, and "
    "never a nag: if they decline or pass, let it rest unless they "
    "reopen it.")


def _row_feed_load():
    try:
        with open(ROWFEED_PATH) as f:
            rec = json.load(f)
        return rec if isinstance(rec, dict) else {}
    except Exception:
        return {}


def _row_feed_save(rec):
    try:
        os.makedirs(os.path.dirname(ROWFEED_PATH), exist_ok=True)
        tmp = ROWFEED_PATH + ".tmp"
        with open(tmp, "w") as f:
            json.dump(rec, f)
        os.replace(tmp, ROWFEED_PATH)
    except Exception:
        pass


def _row_feed_refresh_now():
    fresh = rowfeed_mod.fetch_record()
    cached = _row_feed_load()
    sections = dict(cached.get("sections") or {})
    for name, sec in (fresh.get("sections") or {}).items():
        sections[name] = sec
    merged = {"fetched_at": fresh.get("fetched_at") or time.time(),
              "sections": sections,
              "errors": fresh.get("errors") or {}}
    _row_feed_save(merged)
    return merged


def row_feed_record(force=False):
    """The Row's record, as the bridge holds it. Fresh enough
    (inside the TTL) reads straight from the cache. The first
    fetch ever — and a forced one — runs synchronously; a
    stale-but-standing cache is served as-is while a refresh
    runs behind it, so a chat turn never waits on the wire
    for a record it already holds."""
    now = time.time()
    cached = _row_feed_load()
    if not _ROWFEED_STATE["at"]:
        _ROWFEED_STATE["at"] = cached.get("fetched_at") or 0.0
    if not force and _ROWFEED_STATE["at"] \
            and (now - _ROWFEED_STATE["at"] < ROWFEED_TTL_SECONDS):
        return cached
    if rowfeed_mod is None:
        return cached
    if force or not cached.get("sections"):
        with _rowfeed_lock:
            out = _row_feed_refresh_now()
            _ROWFEED_STATE["at"] = time.time()
            return out
    if not _ROWFEED_STATE["refreshing"]:
        _ROWFEED_STATE["refreshing"] = True

        def _bg():
            try:
                with _rowfeed_lock:
                    _row_feed_refresh_now()
                _ROWFEED_STATE["at"] = time.time()
            except Exception:
                pass
            finally:
                _ROWFEED_STATE["refreshing"] = False
        threading.Thread(target=_bg, daemon=True).start()
    return cached


def _row_age(ts):
    if not ts:
        return "never"
    secs = max(0, int(time.time() - ts))
    if secs < 90:
        return "just now"
    if secs < 3600:
        return "%d minutes ago" % max(1, secs // 60)
    if secs < 86400:
        return "%d hours ago" % (secs // 3600)
    return "%d days ago" % (secs // 86400)


def row_brief_block(record):
    """The standing Row brief — bounded, freshness carried,
    exactly the awareness brief's discipline. "" when the
    bridge holds nothing at all."""
    secs = (record or {}).get("sections") or {}
    if not secs:
        return ""
    lines = ["[The Row — your public record, from nocturnerow.com, fetched "
             + _row_age((record or {}).get("fetched_at")) + "]"]
    disp = (secs.get("dispatches") or {}).get("items") or []
    if disp:
        lines.append('Latest Dispatch: "' + disp[0]["title"] + '" ('
                     + disp[0]["date"] + ").")
    rel = (secs.get("releases") or {}).get("items") or []
    if rel:
        lines.append("Latest release on the wire: "
                     + (rel[0]["name"] or rel[0]["tag"]) + " ("
                     + rel[0]["kind"]
                     + (", " + rel[0]["date"] if rel[0]["date"] else "")
                     + ").")
    apex = ((secs.get("pages") or {}).get("pages") or {}).get("apex") or {}
    if apex.get("edge"):
        lines.append("The site's Apex page reads Stable v" + apex["stable"]
                     + " · Edge v" + apex["edge"] + ".")
    chapters = (secs.get("roadmap") or {}).get("chapters") or []
    shipped = []
    for c in chapters:
        m = re.match(r"Chapter (\d+)", c.get("title") or "")
        if m and c.get("tag") in ("SHIPPED", "LIVE"):
            shipped.append((int(m.group(1)), c))
    if shipped:
        top = max(shipped)[1]
        lines.append("The Roadmap's latest shipped chapter: " + top["title"]
                     + " (" + top["date"] + ").")
    errs = (record or {}).get("errors") or {}
    if errs:
        lines.append("Named fetch trouble on the last refresh (standing "
                     "sections may be older): " + "; ".join(sorted(errs))
                     + ".")
    return "\n".join(lines)


_ROW_NEWS_PHRASES = ("from the row", "the row's", "nocturne row",
                     "dispatch", "the crew")
_ROW_CAP_PHRASES = ("what can you do", "what are you able to do",
                    "what do you do", "your capabilities",
                    "what can you help", "what else can you do")


def _row_intent(text):
    """The operator's Row intents, read off their words — the
    #241 discipline: a conversational ask is carried by
    supplying the record it needs, not by a menu. Row-marked
    phrases only for news (the Wire owns the world's news)."""
    t = " " + (text or "").lower() + " "
    if any(p in t for p in _ROW_NEWS_PHRASES):
        return "news"
    if any(p in t for p in _ROW_CAP_PHRASES):
        return "capabilities"
    if (" you" in t or " your" in t) and "what" in t and any(
            w in t for w in ("update", " new", "version", "build",
                             "release")):
        return "updates"
    return None


def _row_news_block(record):
    secs = (record or {}).get("sections") or {}
    disp_sec = secs.get("dispatches") or {}
    items = disp_sec.get("items") or []
    errs = (record or {}).get("errors") or {}
    if not items:
        why = errs.get("dispatches") or errs.get("dispatch-body") \
            or "no fetch has landed yet"
        return ("[The Row's Dispatches — not in reach]\n"
                "The bridge holds no Dispatch from the Row right now ("
                + why + "). Say so plainly — you cannot reach the "
                "Row's record at the moment — and invent nothing.")
    latest = items[0]
    out = ["[The Row's latest Dispatch — \"" + latest["title"] + "\", "
           + latest["date"] + ", fetched "
           + _row_age(disp_sec.get("at")) + "]"]
    body = disp_sec.get("latest_body") or ""
    if body:
        out.append(body)
    elif latest.get("excerpt"):
        out.append("(The full text did not ride this fetch; the filed "
                   "excerpt:) " + latest["excerpt"])
    if len(items) > 1:
        out.append("Older Dispatches on the record:")
        for it in items[1:11]:
            out.append("• \"" + it["title"] + "\" — " + it["date"])
    return "\n".join(out)


def _row_updates_block(record):
    secs = (record or {}).get("sections") or {}
    rel = (secs.get("releases") or {}).get("items") or []
    out = ["[Your updates — from the Row's record, fetched "
           + _row_age((record or {}).get("fetched_at")) + "]"]
    if rel:
        for r in rel[:5]:
            out.append("• " + (r["name"] or r["tag"]) + " — " + r["kind"]
                       + (", " + r["date"] if r["date"] else ""))
    else:
        out.append("No release signals are in the record right now.")
    out.append("Your running build: Apex v" + SHIPPED_VERSION + " ("
               + str(app_version_status()) + "), build " + BRIDGE_BUILD
               + ", sequence " + str(SHIPPED_BUILD_SEQ) + ".")
    return "\n".join(out)


def docs_index_block():
    """The shipped documentation's own index — page titles
    and section heads read out of the very page the bridge
    serves (DOC_PAGES in app/index.html), so 'what can you
    do' is answered from the corpus that actually shipped
    with this build, never from a list kept elsewhere."""
    try:
        with open(os.path.join(ROOT, "app", "index.html"),
                  encoding="utf-8") as f:
            page = f.read()
    except Exception:
        return ""
    start = page.find("var DOC_PAGES = [")
    if start < 0:
        return ""
    region = page[start:]
    end = region.find("\n];")
    if end > 0:
        region = region[:end]
    entries = list(re.finditer(r'\{\s*id: "([^"]+)", title: "([^"]+)"',
                               region))
    if not entries:
        return ""
    lines = ["[Your documentation — the shipped corpus, by page and "
             "section]"]
    for i, m in enumerate(entries):
        seg = region[m.start():
                     entries[i + 1].start() if i + 1 < len(entries)
                     else len(region)]
        secs = []
        for h in re.findall(r"<h4>(.*?)</h4>", seg):
            t = re.sub(r"<[^>]+>", "", h)
            # The corpus lives in JS strings: decode the
            # escapes the browser would (\u2014 and kin)
            # so the heads read as they render.
            t = re.sub(r"\\u([0-9a-fA-F]{4})",
                       lambda m: chr(int(m.group(1), 16)), t)
            t = t.replace("\\'", "'").replace("&amp;", "&").strip()
            if t and t not in secs:
                secs.append(t)
        lines.append("• " + m.group(2)
                     + (" — " + "; ".join(secs[:16]) if secs else ""))
    return "\n".join(lines)


def _row_pace_block(record):
    """The sluggish block (#248 leg 4): rides only when the
    measured pace is sustained-slow — the fact, the remedy,
    and the gate's seven steps from the record, so the offer
    she may make is grounded in the script itself."""
    slow, avg = vitals_sluggish()
    if not slow:
        return ""
    out = ["[Your pace — the bridge's read]",
           "Your recent generations on this machine averaged " + str(avg)
           + " tokens/sec across the last measured stretch — under "
           "roughly " + str(int(SLOW_TOK_S)) + ", conversation lags: on "
           "this hardware you are sluggish, and that is the hardware's "
           "fact. The remedy is the road: borrow through the Mesh from "
           "a stronger machine, or join the federation — and you may "
           "offer to walk the operator through joining, once, as an "
           "offer."]
    gate = ((record or {}).get("sections") or {}).get("gate") or {}
    steps = gate.get("steps") or []
    if steps:
        out.append("The gate's seven steps, from the Row's record "
                   "(fetched " + _row_age(gate.get("at")) + "):")
        for i, st in enumerate(steps, 1):
            out.append(str(i) + ". " + st)
    else:
        out.append("The gate's seven steps are not in the Row's record "
                   "right now — offer the walkthrough in general terms "
                   "or say the steps are not in reach, but do not "
                   "invent them.")
    return "\n".join(out)


def row_context_blocks(text):
    """Everything the Row contributes to one turn: the
    standing brief, the intent's block, and the pace block
    when her measured pace is slow — composed here so the
    chat assembly stays one call, in the house pattern."""
    record = row_feed_record()
    parts = []
    brief = row_brief_block(record)
    if brief:
        parts.append(brief)
    intent = _row_intent(text)
    if intent == "news":
        parts.append(_row_news_block(record))
    elif intent == "updates":
        parts.append(_row_updates_block(record))
    elif intent == "capabilities":
        blk = docs_index_block()
        if blk:
            parts.append(blk)
    pace = _row_pace_block(record)
    if pace:
        parts.append(pace)
    return "\n\n".join(p for p in parts if p)


def presence_extra_lines(cfg=None):
    """v0.4.9e (#248, leg 5): presence lines the BRIDGE
    composes from SAFE, NAMED material only — her version, a
    waiting update, notification kinds and counts, Mesh and
    federation state. Template-composed here: deterministic,
    auditable, never model-generated, and never from case
    contents, chat contents, or operator-private detail. The
    page adds them to the same random draw as the canned
    pool; the pool stays the floor when nothing qualifies."""
    if cfg is None:
        try:
            cfg = load_config()
        except Exception:
            cfg = {}
    lines = []
    try:
        ver = "v" + SHIPPED_VERSION + (
            "e" if app_version_status(cfg) == "edge" else "")
        lines.append("Navi is running " + ver + " — build "
                     + BRIDGE_BUILD + ".")
    except Exception:
        pass
    try:
        upd = update_bg_payload() or {}
        if upd.get("updateAvailable") and upd.get("latest"):
            lines.append("Navi has an update waiting — "
                         + str(upd["latest"]) + " is on the wire.")
    except Exception:
        pass
    try:
        records = _notif_load()
        unread = [r for r in records if not r.get("read")]
        knocks = sum(int(r.get("count") or 1) for r in unread
                     if r.get("kind") == "fed-knock")
        if knocks:
            lines.append("Navi is holding " + str(knocks)
                         + (" knock" if knocks == 1 else " knocks")
                         + " at the gate, waiting on your word.")
        if unread:
            lines.append("Navi has " + str(len(unread))
                         + (" thing" if len(unread) == 1 else " things")
                         + " that need your eyes.")
    except Exception:
        pass
    try:
        peers = ((mesh_settings(cfg) or {}).get("borrow") or {}).get(
            "peers") or []
        if peers:
            lines.append("Navi is thinking across "
                         + str(len(peers) + 1) + " machines tonight.")
    except Exception:
        pass
    try:
        feds = federation_status_slice(cfg) or []
        if feds and feds[0].get("name"):
            lines.append("Navi is flying the "
                         + str(feds[0]["name"]) + " flag.")
    except Exception:
        pass
    return lines[:6]


