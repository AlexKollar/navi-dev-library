# Satellite modules

## rowfeed.py

Source: `rowfeed.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

#!/usr/bin/env python3
# rowfeed.py — the Row's public record, fetched and parsed
# (v0.4.9e, #248, Context Awareness leg 2).
#
# Apex and nocturnerow.com are one ecosystem, and this module
# is the bridge's read path into the Row's half of it: the
# Dispatches, the Roadmap's chapters, her own page and the
# Apex page, the release signals, and the gate's seven steps.
# The site publishes no feed, JSON, or sitemap for these (the
# checks were made: /feed.xml and /sitemap.xml answer 404),
# so the record is parsed from the public HTML itself — the
# one structured exception is the release wire, which is the
# same GitHub releases JSON the Apex page's own Signals
# section reads as you open it. The site's wire is the wire.
#
# Everything here is pure: fetchers are injected, parsers take
# text and return data, and no function keeps state. The
# bridge owns the cache (config/row-feed.json), the refresh
# throttle, and the honesty rules — a section that fails to
# fetch keeps its last good data in the cache and its error
# is NAMED in the record, never silently emptied.
import html as _html
import json
import re
import time
import urllib.request

SITE = "https://nocturnerow.com"
BLOG_URL = SITE + "/blog.html"
ROADMAP_URL = SITE + "/roadmap.html"
APEX_URL = SITE + "/apex.html"
NAVI_URL = SITE + "/navi.html"
GATE_URL = SITE + "/rockport-mesh.html"
RELEASES_URL = ("https://api.github.com/repos/AlexKollar/Apex"
                "/releases?per_page=5")

BODY_CAP = 6000


def _fetch(url, timeout=25):
    # The wire.py/_fetch idiom: one plain GET, an honest
    # user agent, no retries hidden from the caller — a
    # failure raises and the bridge names it.
    req = urllib.request.Request(url, headers={
        "User-Agent": "Apex-RowFeed/1.0 (local-first; operator machine)",
        "Accept": "text/html, application/json",
    })
    with urllib.request.urlopen(req, timeout=timeout) as resp:
        return resp.read().decode("utf-8", "replace")


def _text(fragment):
    # HTML fragment -> plain text: scripts and styles out,
    # tags out, entities unescaped, whitespace collapsed.
    s = re.sub(r"(?is)<(script|style)[^>]*>.*?</\1>", " ", fragment or "")
    s = re.sub(r"(?s)<[^>]+>", " ", s)
    s = _html.unescape(s)
    return re.sub(r"\s+", " ", s).strip()


# -- Dispatches ---------------------------------------------------------

_CARD_RE = re.compile(r'(?s)<article class="post">(.*?)</article>')
_TITLE_RE = re.compile(r'(?s)<h3 class="ptitle"><a href="([^"]+)">(.*?)</a></h3>')
_META_RE = re.compile(r'(?s)<div class="meta">(.*?)</div>')
_EXCERPT_RE = re.compile(r'(?s)<p class="excerpt">(.*?)</p>')
_DTG_RE = re.compile(r'TRANSMISSION //\s*([0-9A-Z ]+)')


def parse_dispatch_index(page_html):
    """The Dispatches index -> [{title, date, dtg, url,
    excerpt}], newest first, exactly as the page files them."""
    out = []
    for card in _CARD_RE.findall(page_html or ""):
        t = _TITLE_RE.search(card)
        if not t:
            continue
        meta = _META_RE.search(card)
        exc = _EXCERPT_RE.search(card)
        dtg = _DTG_RE.search(card)
        out.append({
            "title": _text(t.group(2)),
            "url": SITE + t.group(1) if t.group(1).startswith("/") else t.group(1),
            "date": (_text(meta.group(1)).split("·")[0].strip()
                     if meta else ""),
            "dtg": dtg.group(1).strip() if dtg else "",
            "excerpt": _text(exc.group(1)) if exc else "",
        })
    return out


def parse_dispatch_body(page_html):
    """One Dispatch's page -> its body as plain text (the
    article if the layout gives one, else the main column),
    bounded by BODY_CAP at the caller's request."""
    m = re.search(r"(?s)<article[^>]*>(.*?)</article>", page_html or "")
    if not m:
        m = re.search(r"(?s)<main[^>]*>(.*?)</main>", page_html or "")
    return _text(m.group(1) if m else (page_html or ""))


# -- The Roadmap ----------------------------------------------------------

_CH_RE = re.compile(
    r'(?s)<span class="tl-date[^"]*">(.*?)</span>\s*'
    r'<span class="tl-title[^"]*">(.*?)</span>\s*'
    r'<span class="tl-tag[^"]*">(.*?)</span>')


def parse_roadmap(page_html):
    """The Roadmap -> [{title, date, tag}] in the page's own
    order — the story of record, chapter by chapter."""
    out = []
    for date, title, tag in _CH_RE.findall(page_html or ""):
        tag = _text(tag)
        if tag.startswith("SHIPPED"):
            tag = "SHIPPED"
        elif tag.startswith("LIVE"):
            tag = "LIVE"
        elif "HORIZON" in tag:
            tag = "HORIZON"
        out.append({"title": _text(title), "date": _text(date),
                    "tag": tag})
    return out


# -- The pages ------------------------------------------------------------

_PILL_RE = re.compile(
    r'Apex <b>v([^<]+) Stable</b>\s*·\s*Edge <b>v([^<]+)</b>')
_DESC_RE = re.compile(r'(?s)<meta name="description" content="([^"]*)"')
_TITLE_TAG_RE = re.compile(r"(?s)<title>(.*?)</title>")


def parse_apex_page(page_html):
    """The Apex page -> {stable, edge} from its version pill,
    the facts the page itself leads with."""
    m = _PILL_RE.search(page_html or "")
    return {"stable": m.group(1) if m else "",
            "edge": m.group(2) if m else ""}


def parse_navi_page(page_html):
    """Her own page -> {title, description}: who the site
    says she is, in the site's own words."""
    t = _TITLE_TAG_RE.search(page_html or "")
    d = _DESC_RE.search(page_html or "")
    return {"title": _text(t.group(1)) if t else "",
            "description": _html.unescape(d.group(1)).strip() if d else ""}


# -- The gate -------------------------------------------------------------

def parse_gate_steps(page_html):
    """The gate page -> the seven join steps, as plain text,
    from the 'How to connect' ordered list — the walkthrough
    script, sourced from the page itself, never retyped."""
    m = re.search(r'(?s)<div class="join-body">\s*<ol>(.*?)</ol>', page_html or "")
    if not m:
        return []
    return [_text(li) for li in re.findall(r"(?s)<li>(.*?)</li>", m.group(1))]


# -- Release signals --------------------------------------------------------

def parse_releases(json_text):
    """The releases wire (the same JSON the Apex page's
    Signals section renders) -> [{tag, name, kind, date,
    excerpt, url}], newest first as the wire files them."""
    try:
        rows = json.loads(json_text or "[]")
    except Exception:
        return []
    out = []
    for r in rows if isinstance(rows, list) else []:
        if not isinstance(r, dict):
            continue
        body = re.sub(r"\s+", " ", re.sub(r"[#>*`_]", "", r.get("body") or "")).strip()
        out.append({
            "tag": r.get("tag_name") or "",
            "name": r.get("name") or r.get("tag_name") or "",
            "kind": "Edge" if r.get("prerelease") else "Stable",
            "date": (r.get("published_at") or "")[:10],
            "excerpt": body[:300],
            "url": r.get("html_url") or "",
        })
    return out


# -- The whole record -------------------------------------------------------

def fetch_record(fetcher=None):
    """One refresh of the whole record. Every source is
    fetched independently: a source that answers lands in
    sections with its own fetch time; a source that fails is
    NAMED in errors and contributes nothing — the bridge's
    merge keeps the cached section standing (see bridge.py).
    The latest Dispatch's body rides along (bounded) so a
    news answer never needs a second fetch."""
    fetch = fetcher or _fetch
    now = time.time()
    sections, errors = {}, {}

    def take(name, url, parse, key):
        try:
            data = parse(fetch(url))
            sections[name] = {"at": now, key: data}
        except Exception as e:
            errors[name] = "fetch-failed: %s: %s" % (type(e).__name__, e)

    take("dispatches", BLOG_URL, parse_dispatch_index, "items")
    take("roadmap", ROADMAP_URL, parse_roadmap, "chapters")
    take("releases", RELEASES_URL, parse_releases, "items")
    take("gate", GATE_URL, parse_gate_steps, "steps")
    pages = {}
    try:
        pages["apex"] = parse_apex_page(fetch(APEX_URL))
        pages["navi"] = parse_navi_page(fetch(NAVI_URL))
        sections["pages"] = {"at": now, "pages": pages}
    except Exception as e:
        errors["pages"] = "fetch-failed: %s: %s" % (type(e).__name__, e)
    disp = sections.get("dispatches", {}).get("items") or []
    if disp:
        try:
            body = parse_dispatch_body(fetch(disp[0]["url"]))
            sections["dispatches"]["latest_body"] = body[:BODY_CAP]
        except Exception as e:
            errors["dispatch-body"] = "fetch-failed: %s: %s" % (
                type(e).__name__, e)
    return {"fetched_at": now, "sections": sections, "errors": errors}


## bounties.py

Source: `bounties.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# bounties.py — the bug-bounty board engine (0.2.0e Build 7,
# Scope Addition 5).
#
# Programs from ProjectDiscovery's Chaos index — the public,
# keyless JSON of public bug-bounty programs. Verified live at
# bake time (2026-11-13): the index moved off the marketing
# host (chaos.projectdiscovery.io/index.json answers 404) and
# lives on the data host:
#   https://chaos-data.projectdiscovery.io/index.json
# Entries carry: name, program_url (the program link),
# platform, bounty (pays-money flag), count (the program's
# enumerated scope size in the Chaos dataset), is_new,
# last_updated. The index carries NO safe-harbor field — cards
# say "not stated by the index" rather than inventing one.
#
# Disclosed reports: HackerOne's hacktivity was probed live at
# bake time and no keyless feed proved out (the GraphQL
# endpoint answers, but exposes no public hacktivity field
# without an account). Per the build's ruling, the family
# ships programs-first with the reports section as a NAMED
# empty state — a sentence, never invented rows. The store and
# payload keep a reports slot so a proven feed can land later
# without a shape change.
#
# Same posture as the wire: outbound-only, no accounts, no
# keys, local cache (config/bounties/bounties.json), named
# staleness, a dead source is a sentence. Stdlib only; never
# imports the bridge; the fetcher is injectable for fixtures.

import gzip
import json
import os
import threading
import time
import urllib.request
import urllib.error

CHAOS_URL = "https://chaos-data.projectdiscovery.io/index.json"
MAX_PROGRAMS = 3000
REPORTS_SENTENCE = ("No keyless disclosed-reports feed proved out at bake time — "
                    "HackerOne's hacktivity needs an account, so this section "
                    "ships empty on purpose rather than inventing rows.")


def _fetch(url, state, timeout=30):
    req = urllib.request.Request(url, headers={
        "User-Agent": "Apex-Wire/1.0 (local-first news wire; operator machine)",
        "Accept-Encoding": "gzip",
    })
    if state.get("etag"):
        req.add_header("If-None-Match", state["etag"])
    if state.get("lastmod"):
        req.add_header("If-Modified-Since", state["lastmod"])
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            raw = resp.read()
            if resp.headers.get("Content-Encoding") == "gzip":
                raw = gzip.decompress(raw)
            return 200, raw, {
                "etag": resp.headers.get("ETag") or state.get("etag") or "",
                "lastmod": resp.headers.get("Last-Modified") or state.get("lastmod") or "",
            }
    except urllib.error.HTTPError as e:
        if e.code == 304:
            return 304, b"", state
        raise


def normalize_program(p):
    if not isinstance(p, dict):
        return None
    name = str(p.get("name") or "").strip()
    link = str(p.get("program_url") or "").strip()
    if not name or not link:
        return None
    try:
        scope = int(p.get("count") or 0)
    except Exception:
        scope = 0
    return {
        "name": name,
        "platform": str(p.get("platform") or "").strip(),
        "link": link,
        "bounty": bool(p.get("bounty")),
        "safeHarbor": None,  # the index does not state it; the card says so
        "scopeSize": scope,
        "isNew": bool(p.get("is_new")),
        "lastUpdated": str(p.get("last_updated") or ""),
        "dataUrl": str(p.get("URL") or ""),
    }


def parse_programs(raw):
    data = json.loads(raw.decode("utf-8", "replace"))
    if isinstance(data, dict):
        data = data.get("programs") or []
    out = []
    for p in data if isinstance(data, list) else []:
        norm = normalize_program(p)
        if norm:
            out.append(norm)
    # Bounty-paying first, then the freshly changed, then name.
    out.sort(key=lambda p: (not p["bounty"], p["name"].lower()))
    return out[:MAX_PROGRAMS]


class Bounties:
    def __init__(self, root, settings_getter, fetch=None):
        self.root = root
        self._settings = settings_getter
        self._fetch = fetch or _fetch
        self._lock = threading.Lock()
        self._pulling = False
        self.dir = os.path.join(root, "config", "bounties")
        self.path = os.path.join(self.dir, "bounties.json")

    def _load(self):
        try:
            with open(self.path, encoding="utf-8") as f:
                data = json.load(f)
            if isinstance(data, dict):
                data.setdefault("programs", [])
                data.setdefault("reports", [])
                return data
        except Exception:
            pass
        return {"programs": [], "reports": []}

    def _save(self, data):
        try:
            os.makedirs(self.dir, exist_ok=True)
            tmp = self.path + ".tmp"
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(data, f)
            os.replace(tmp, self.path)
        except Exception:
            pass

    def settings(self):
        return self._settings() or {}

    def refresh_now(self):
        # Threaded like the Center's: the caller never waits
        # on a feed; state() carries the outcome when it lands.
        with self._lock:
            if self._pulling:
                return {"ok": True, "pulling": True}
            self._pulling = True

        def run():
            try:
                self._refresh_cycle()
            finally:
                with self._lock:
                    self._pulling = False

        threading.Thread(target=run, daemon=True).start()
        return {"ok": True, "started": True}

    def _refresh_cycle(self):
        now = int(time.time())
        data = self._load()
        state = {"etag": data.get("etag") or "", "lastmod": data.get("lastmod") or ""}
        try:
            status, raw, new_state = self._fetch(CHAOS_URL, state)
            programs = list(data.get("programs") or []) if status == 304 \
                else parse_programs(raw)
            data = {"programs": programs, "reports": list(data.get("reports") or []),
                    "lastTry": now, "lastOk": now, "error": "",
                    "etag": new_state.get("etag", ""), "lastmod": new_state.get("lastmod", "")}
            self._save(data)
            return {"ok": True, "programs": len(programs)}
        except Exception as e:
            data["lastTry"] = now
            data["error"] = ("The Chaos index could not be fetched (" + str(e)[:140] + ").")
            self._save(data)
            return {"ok": False, "error": data["error"]}

    def start(self):
        def loop():
            while True:
                try:
                    s = self.settings()
                    if s.get("enabled", True):
                        data = self._load()
                        last = data.get("lastTry") or 0
                        interval = max(30, int(s.get("intervalMinutes") or 360)) * 60
                        if time.time() - last >= interval:
                            self.refresh_now()
                except Exception:
                    pass
                time.sleep(60)
        threading.Thread(target=loop, daemon=True).start()

    def staleness_sentence(self, data=None):
        data = data or self._load()
        last_ok = data.get("lastOk") or 0
        if data.get("error"):
            base = "The bounty board's source is not answering: " + data["error"]
            if last_ok:
                base += " The programs below are the last good fetch, from " + \
                    time.strftime("%Y-%m-%d %H:%M", time.gmtime(last_ok)) + " UTC."
            return base
        if not last_ok:
            return "The bounty board has not fetched the Chaos index yet."
        age_h = round((time.time() - last_ok) / 3600.0, 1)
        when = "under an hour ago" if age_h < 1 else str(age_h) + " hours ago"
        return "The Chaos index last fetched cleanly " + when + "."

    def state(self):
        s = self.settings()
        data = self._load()
        programs = list(data.get("programs") or [])
        return {
            "ok": True,
            "enabled": bool(s.get("enabled", True)),
            "intervalMinutes": int(s.get("intervalMinutes") or 360),
            "pulling": self._pulling,
            "lastOk": data.get("lastOk") or 0,
            "staleness": self.staleness_sentence(data),
            "error": data.get("error") or "",
            "programs": programs,
            "payingCount": sum(1 for p in programs if p.get("bounty")),
            "reports": list(data.get("reports") or []),
            "reportsSentence": REPORTS_SENTENCE,
        }

    def brief_block(self, limit=1000):
        s = self.settings()
        if not s.get("enabled", True):
            return ""
        st = self.state()
        if not st["programs"] and not st["lastOk"]:
            return ""
        lines = ["[On the bounty board — public programs from ProjectDiscovery's Chaos index]",
                 self.staleness_sentence(),
                 str(len(st["programs"])) + " programs listed, " + str(st["payingCount"])
                 + " of them bounty-paying."]
        fresh = [p for p in st["programs"] if p.get("isNew")][:4]
        for p in fresh:
            lines.append("- NEW: " + p["name"] + " (" + (p["platform"] or "platform unlisted")
                         + ", " + ("pays bounties" if p["bounty"] else "no bounty listed") + ").")
        lines.append("When the operator asks about bounties, speak from this board and cite "
                     "the Chaos index with its fetch stamp. The index does not state "
                     "safe-harbor terms — say so rather than guessing a program's terms.")
        return "\n".join(lines)[:limit]


## ctf.py

Source: `ctf.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# ctf.py — the CTF board engine (0.2.0e Build 7, #189).
#
# A CTFtime API fetcher + local cache, same posture as the
# wire: outbound-only, no accounts, no keys, named staleness, a
# dead source is a sentence. The board carries everything the
# API gives per event — name, start/finish, format, online or
# on-site + location, weight, restrictions, organizers, logo,
# description, prizes where listed, the CTFtime link. Running
# now and upcoming are the two sections; past events fall off.
# Times ride as epochs in the payload and are rendered in the
# operator's own timezone by the page.
#
# CTFtime refuses requests without a User-Agent (403); the
# fetcher sends Apex's, as the Center's does.
#
# Stdlib only; never imports the bridge; the fetcher is
# injectable so the benches run on a recorded fixture.

import gzip
import html as html_mod
import json
import os
import re
import threading
import time
import urllib.request
import urllib.error
from datetime import datetime, timezone

API_URL = "https://ctftime.org/api/v1/events/"
FORMAT_LABELS = {1: "Jeopardy", 2: "Attack-Defense"}
DESC_CAP = 4000
_TAG_RE = re.compile(r"<[^>]+>")


def _fetch(url, state, timeout=25):
    req = urllib.request.Request(url, headers={
        "User-Agent": "Apex-Wire/1.0 (local-first news wire; operator machine)",
        "Accept-Encoding": "gzip",
    })
    if state.get("etag"):
        req.add_header("If-None-Match", state["etag"])
    if state.get("lastmod"):
        req.add_header("If-Modified-Since", state["lastmod"])
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            raw = resp.read()
            if resp.headers.get("Content-Encoding") == "gzip":
                raw = gzip.decompress(raw)
            return 200, raw, {
                "etag": resp.headers.get("ETag") or state.get("etag") or "",
                "lastmod": resp.headers.get("Last-Modified") or state.get("lastmod") or "",
            }
    except urllib.error.HTTPError as e:
        if e.code == 304:
            return 304, b"", state
        raise


def _epoch(value):
    if isinstance(value, (int, float)):
        return int(value)
    if not value:
        return 0
    try:
        dt = datetime.fromisoformat(str(value).replace("Z", "+00:00"))
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        return int(dt.timestamp())
    except Exception:
        return 0


def _plain(raw_html, cap=DESC_CAP):
    if not raw_html:
        return ""
    text = _TAG_RE.sub(" ", str(raw_html))
    text = html_mod.unescape(text)
    return re.sub(r"\s+", " ", text).strip()[:cap]


def normalize_event(ev):
    if not isinstance(ev, dict):
        return None
    start = _epoch(ev.get("start"))
    finish = _epoch(ev.get("finish"))
    title = str(ev.get("title") or "").strip()
    if not title or not start:
        return None
    fmt = str(ev.get("format") or "").strip()
    if not fmt:
        fmt = FORMAT_LABELS.get(ev.get("format_id"), "")
    organizers = []
    for org in ev.get("organizers") or []:
        if isinstance(org, dict) and org.get("name"):
            organizers.append(str(org["name"]))
        elif isinstance(org, str) and org:
            organizers.append(org)
    try:
        weight = float(ev.get("weight") or 0)
    except Exception:
        weight = 0.0
    return {
        "id": ev.get("id"),
        "title": title,
        "start": start,
        "finish": finish,
        "format": fmt,
        "onsite": bool(ev.get("onsite")),
        "location": str(ev.get("location") or ""),
        "weight": weight,
        "restrictions": str(ev.get("restrictions") or ""),
        "organizers": organizers,
        "logo": str(ev.get("logo") or ""),
        "description": _plain(ev.get("description") or ""),
        "prizes": _plain(ev.get("prizes") or "", cap=1200),
        "url": str(ev.get("url") or ""),
        "ctftimeUrl": str(ev.get("ctftime_url") or ""),
        "participants": ev.get("participants"),
    }


def parse_events(raw):
    data = json.loads(raw.decode("utf-8", "replace"))
    if isinstance(data, dict):
        data = data.get("events") or data.get("results") or []
    out = []
    for ev in data if isinstance(data, list) else []:
        norm = normalize_event(ev)
        if norm:
            out.append(norm)
    out.sort(key=lambda e: e["start"])
    return out


class Ctf:
    def __init__(self, root, settings_getter, fetch=None):
        self.root = root
        self._settings = settings_getter
        self._fetch = fetch or _fetch
        self._lock = threading.Lock()
        self._pulling = False
        self.dir = os.path.join(root, "config", "ctf")
        self.path = os.path.join(self.dir, "events.json")

    def _load(self):
        try:
            with open(self.path, encoding="utf-8") as f:
                data = json.load(f)
            if isinstance(data, dict):
                data.setdefault("events", [])
                return data
        except Exception:
            pass
        return {"events": []}

    def _save(self, data):
        try:
            os.makedirs(self.dir, exist_ok=True)
            tmp = self.path + ".tmp"
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(data, f)
            os.replace(tmp, self.path)
        except Exception:
            pass

    def settings(self):
        return self._settings() or {}

    def refresh_now(self):
        # Threaded like the Center's: the caller never waits
        # on a feed; state() carries the outcome when it lands.
        with self._lock:
            if self._pulling:
                return {"ok": True, "pulling": True}
            self._pulling = True

        def run():
            try:
                self._refresh_cycle()
            finally:
                with self._lock:
                    self._pulling = False

        threading.Thread(target=run, daemon=True).start()
        return {"ok": True, "started": True}

    def _refresh_cycle(self):
        now = int(time.time())
        url = (API_URL + "?limit=100&start=" + str(now - 2 * 86400)
               + "&finish=" + str(now + 60 * 86400))
        data = self._load()
        try:
            status, raw, _ns = self._fetch(url, {})
            events = list(data.get("events") or []) if status == 304 \
                else parse_events(raw)
            data = {"events": events, "lastTry": now, "lastOk": now,
                    "error": ""}
            self._save(data)
            return {"ok": True, "events": len(events)}
        except Exception as e:
            data["lastTry"] = now
            data["error"] = ("CTFtime could not be fetched (" + str(e)[:140] + ").")
            self._save(data)
            return {"ok": False, "error": data["error"]}

    def start(self):
        def loop():
            while True:
                try:
                    s = self.settings()
                    if s.get("enabled", True):
                        data = self._load()
                        last = data.get("lastTry") or 0
                        interval = max(10, int(s.get("intervalMinutes") or 60)) * 60
                        if time.time() - last >= interval:
                            self.refresh_now()
                except Exception:
                    pass
                time.sleep(60)
        threading.Thread(target=loop, daemon=True).start()

    def staleness_sentence(self, data=None):
        data = data or self._load()
        last_ok = data.get("lastOk") or 0
        if data.get("error"):
            base = "CTFtime is not answering: " + data["error"]
            if last_ok:
                base += " The board below is the last good fetch, from " + \
                    time.strftime("%Y-%m-%d %H:%M", time.gmtime(last_ok)) + " UTC."
            return base
        if not last_ok:
            return "The CTF board has not fetched yet."
        age_min = int((time.time() - last_ok) / 60)
        when = ("under a minute ago" if age_min < 1
                else str(age_min) + " minutes ago" if age_min < 90
                else str(round(age_min / 60.0, 1)) + " hours ago")
        return "The board last fetched cleanly " + when + "."

    def state(self):
        s = self.settings()
        data = self._load()
        now = time.time()
        events = [e for e in data.get("events") or [] if (e.get("finish") or 0) >= now]
        running = [e for e in events if e["start"] <= now <= e["finish"]]
        upcoming = [e for e in events if e["start"] > now]
        return {
            "ok": True,
            "enabled": bool(s.get("enabled", True)),
            "intervalMinutes": int(s.get("intervalMinutes") or 60),
            "pulling": self._pulling,
            "lastOk": data.get("lastOk") or 0,
            "staleness": self.staleness_sentence(data),
            "error": data.get("error") or "",
            "running": running,
            "upcoming": upcoming,
        }

    def brief_block(self, limit=1200):
        s = self.settings()
        if not s.get("enabled", True):
            return ""
        st = self.state()
        if not st["running"] and not st["upcoming"] and not st["lastOk"]:
            return ""
        lines = ["[On the CTF board — from CTFtime, via the local cache]",
                 self.staleness_sentence()]
        for e in st["running"][:3]:
            lines.append("- RUNNING NOW: " + e["title"] + " (" + (e["format"] or "format unlisted")
                         + ", " + ("on-site, " + e["location"] if e["onsite"] else "online")
                         + ") — ends " + time.strftime("%Y-%m-%d %H:%M", time.gmtime(e["finish"])) + " UTC.")
        for e in st["upcoming"][:4]:
            lines.append("- " + e["title"] + " — starts "
                         + time.strftime("%Y-%m-%d %H:%M", time.gmtime(e["start"])) + " UTC ("
                         + (e["format"] or "format unlisted") + ", "
                         + ("on-site, " + e["location"] if e["onsite"] else "online")
                         + ", weight " + str(e["weight"]) + ").")
        lines.append("When the operator asks what's on or what's worth playing, speak from "
                     "this board and cite the listing (CTFtime, with its fetch stamp) — the "
                     "board's times here are UTC; the CTF tab shows them in the operator's "
                     "own timezone.")
        return "\n".join(lines)[:limit]
