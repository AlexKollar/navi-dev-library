# bridge.py — part 4 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 12447–16274 of bridge.py.
Top-level symbols in this part: _mesh_effective_addresses, mesh_tunnel_state, _mesh_tunnel_set_state, _mesh_tunnel_classify, _mesh_tunnel_wait, _mesh_tunnel_supervise, _mesh_tunnel_allocate_ports, mesh_tunnel_sync, mesh_tunnel_stop_all, mesh_start_listener, mesh_stop_listener, read_file, app_version_status, app_version, persona_prompt, memory_looks_secret, normalize_memory_fact, memory_fact_key, memory_trigger, extract_memory_markers, expand, _is_sudo_noise, clean_sudo_output, sudo_settings, _sudo_probe, sudo_state, sudo_arm, sudo_disarm, sudo_audit_append, sudo_audit_entries, path_complete, run_shell, _strip_ansi, _TermGrid, _key_display, ShellSession, _shell_sweeper, _ensure_sweeper, shell_session_open, shell_session_get, shell_session_input, shell_session_close, _shell_session_died, shell_session_state, shell_sessions_kill_all, shell_machine_intent, shell_turn_machine_directed, _shell_line_prose, extract_shell_block, _strip_shell_blocks, prose_guard, shell_session_brief, installed_commands, known_commands, cli_route, GithubHttpError, _gh_header, _github_note_rate, github_rate_state, _gh_reset_phrase, github_classify_error, github_error_info, github_error_text, _now_iso, _github_read_headers, _github_read, _github_read_once, github_json, github_text, github_search_repositories, _version_tuple, _version_newer, _status_rank, _release_is_newer, _release_build_seq, _update_error_text, updater_channel_explicit, updater_channel, _shape_release, latest_release.


def _mesh_effective_addresses(peer):
    # The dial list: the record's entries in their order, with
    # the managed tunnel endpoint appended LAST — computed here,
    # at dial time; the stored record never carries it.
    addrs = list(peer.get("addresses") or [])
    ent = _mesh_tunnel_entry(peer)
    if ent and ent not in addrs:
        addrs.append(ent)
    return addrs


def mesh_tunnel_state(node_id):
    with MESH_TUNNEL_LOCK:
        st = MESH_TUNNELS.get(node_id)
        if st is not None:
            return {"state": st.get("state") or "down",
                    "sentence": st.get("sentence") or ""}
    return {"state": "off", "sentence": ""}


def _mesh_tunnel_set_state(node_id, state, sentence):
    log = False
    with MESH_TUNNEL_LOCK:
        st = MESH_TUNNELS.get(node_id)
        if st is None:
            return
        if st.get("state") != state or st.get("sentence") != sentence:
            st["state"] = state
            st["sentence"] = sentence
            log = bool(sentence) and state != "starting"
    if log:
        # One line per state change, never per retry.
        print("Navi Mesh: " + sentence, flush=True)


def _mesh_tunnel_classify(stderr_text, name):
    # Conservative reading of the ssh process's last words:
    # only the signatures that unambiguously mean one thing are
    # named; everything else is honestly just "tunnel down".
    s = (stderr_text or "").lower()
    if "permission denied" in s:
        return "unauthorized", (
            "The backup tunnel to " + name + " is down: " + name +
            " has not authorized this machine's tunnel key. On " +
            name + "'s own dashboard, its Mesh page offers to "
            "authorize this machine's tunnel key — once, in "
            "person — and the road keeps itself after that.")
    if any(x in s for x in ("could not resolve",
                            "name or service not known",
                            "connection timed out",
                            "no route to host",
                            "network is unreachable",
                            "connection refused")):
        return "unreachable", (
            "The backup tunnel to " + name + " is down: its SSH "
            "door did not answer from here.")
    return "down", "The backup tunnel to " + name + " is down."


def _mesh_tunnel_wait(node_id, seconds):
    # Sleep in slices; True means the supervisor should exit.
    end = time.time() + max(0.0, seconds)
    while time.time() < end:
        with MESH_TUNNEL_LOCK:
            st = MESH_TUNNELS.get(node_id)
            if st is None or st.get("stop"):
                return True
        time.sleep(0.5)
    with MESH_TUNNEL_LOCK:
        st = MESH_TUNNELS.get(node_id)
        return st is None or bool(st.get("stop"))


def _mesh_tunnel_supervise(node_id):
    fails = 0
    while True:
        with MESH_TUNNEL_LOCK:
            st = MESH_TUNNELS.get(node_id)
            if st is None or st.get("stop"):
                return
        cfg = load_config()
        ms = mesh_settings(cfg)
        peer = next((p for p in ms["borrow"]["peers"]
                     if p["nodeId"] == node_id), None)
        if peer is None or not ms["enabled"] or \
                not (peer.get("tunnel") or {}).get("enabled"):
            with MESH_TUNNEL_LOCK:
                MESH_TUNNELS.pop(node_id, None)
            return
        tun = peer["tunnel"]
        host = tun.get("sshHost") or ""
        user = tun.get("sshUser") or ""
        lp = tun.get("localPort")
        if not host or not user or not lp:
            _mesh_tunnel_set_state(
                node_id, "down",
                "The backup route to " + peer["name"] + " is on but "
                "not filled in: it needs that machine's SSH host "
                "and user, set on the Mesh page.")
            if _mesh_tunnel_wait(node_id, 5):
                return
            continue
        ok, err = mesh_ensure_tunnel_key()
        if not ok:
            _mesh_tunnel_set_state(node_id, "down", err)
            if _mesh_tunnel_wait(node_id, 30):
                return
            continue
        argv = ["ssh", "-N",
                "-L", str(lp) + ":127.0.0.1:" + str(peer["port"]),
                "-i", MESH_TUNNEL_KEY_PATH,
                "-p", str(tun["sshPort"]),
                "-o", "BatchMode=yes",
                "-o", "IdentitiesOnly=yes",
                "-o", "ExitOnForwardFailure=yes",
                "-o", "ServerAliveInterval=30",
                "-o", "ServerAliveCountMax=3",
                "-o", "StrictHostKeyChecking=accept-new",
                user + "@" + host]
        _mesh_tunnel_set_state(node_id, "starting", "")
        try:
            proc = subprocess.Popen(argv, stdout=subprocess.DEVNULL,
                                    stderr=subprocess.PIPE)
        except Exception:
            _mesh_tunnel_set_state(
                node_id, "down",
                "The backup tunnel to " + peer["name"] + " could "
                "not start — the ssh program did not answer on "
                "this machine.")
            if _mesh_tunnel_wait(node_id, 30):
                return
            continue
        with MESH_TUNNEL_LOCK:
            st = MESH_TUNNELS.get(node_id)
            if st is not None:
                st["proc"] = proc
        started = time.time()
        up = False
        while proc.poll() is None:
            with MESH_TUNNEL_LOCK:
                st = MESH_TUNNELS.get(node_id)
                stop = st is None or st.get("stop")
            if stop:
                try:
                    proc.terminate()
                except Exception:
                    pass
                try:
                    proc.wait(timeout=5)
                except Exception:
                    try:
                        proc.kill()
                    except Exception:
                        pass
                return
            if not up and time.time() - started >= 3:
                # ssh exits at once when the forward cannot be
                # established (ExitOnForwardFailure), so a
                # process alive after a few seconds is carrying
                # the road.
                up = True
                fails = 0
                _mesh_tunnel_set_state(
                    node_id, "up",
                    "The backup tunnel to " + peer["name"] + " is "
                    "up: its Mesh door answers here at 127.0.0.1:" +
                    str(lp) + ", carried by its SSH door.")
            time.sleep(0.5)
        err_text = ""
        try:
            err_text = (proc.stderr.read() or b"").decode("utf-8",
                                                          "replace")
        except Exception:
            pass
        word, sentence = _mesh_tunnel_classify(err_text, peer["name"])
        _mesh_tunnel_set_state(node_id, word, sentence)
        backoff = _MESH_TUNNEL_BACKOFFS[min(fails,
                                            len(_MESH_TUNNEL_BACKOFFS) - 1)]
        fails += 1
        if _mesh_tunnel_wait(node_id, backoff):
            return


def _mesh_tunnel_allocate_ports(cfg):
    # Give every route-enabled peer that lacks a local port a
    # stable one, persisted into the stored record once chosen.
    # Candidates are probed free on loopback and kept clear of
    # the door ports and of each other. Returns True when the
    # config changed (the caller reloads).
    ms = mesh_settings(cfg)
    taken = set(ms["listenPorts"])
    for p in ms["borrow"]["peers"]:
        lp = (p.get("tunnel") or {}).get("localPort")
        if lp:
            taken.add(lp)
    block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
    borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
    raw_peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
    changed = False
    for rp in raw_peers:
        if not isinstance(rp, dict):
            continue
        tun = rp.get("tunnel")
        if not isinstance(tun, dict) or tun.get("enabled") is not True:
            continue
        if tun.get("localPort"):
            continue
        for cand in range(18765, 19265):
            if cand in taken:
                continue
            try:
                s = socket.socket()
                s.bind(("127.0.0.1", cand))
                s.close()
            except Exception:
                continue
            tun["localPort"] = cand
            taken.add(cand)
            changed = True
            break
    if changed:
        save_config(cfg)
    return changed


def mesh_tunnel_sync():
    # Reconcile the supervisors with the config: start one for
    # each peer whose backup route is on (allocating its local
    # port first), stop the rest. Called when the Mesh starts,
    # when Mesh settings save, and when a route is edited.
    cfg = load_config()
    ms = mesh_settings(cfg)
    if ms["enabled"] and any((p.get("tunnel") or {}).get("enabled")
                             and not (p.get("tunnel") or {}).get("localPort")
                             for p in ms["borrow"]["peers"]):
        if _mesh_tunnel_allocate_ports(cfg):
            cfg = load_config()
            ms = mesh_settings(cfg)
    want = set()
    if ms["enabled"]:
        for p in ms["borrow"]["peers"]:
            if (p.get("tunnel") or {}).get("enabled"):
                want.add(p["nodeId"])
    with MESH_TUNNEL_LOCK:
        for node_id in list(MESH_TUNNELS.keys()):
            if node_id not in want:
                st = MESH_TUNNELS.pop(node_id)
                st["stop"] = True
                proc = st.get("proc")
                if proc is not None:
                    try:
                        proc.terminate()
                    except Exception:
                        pass
        for node_id in want:
            st = MESH_TUNNELS.get(node_id)
            if st is None or not st.get("thread") or \
                    not st["thread"].is_alive():
                st = {"stop": False, "proc": None, "state": "",
                      "sentence": "", "thread": None}
                t = threading.Thread(target=_mesh_tunnel_supervise,
                                     args=(node_id,), daemon=True)
                st["thread"] = t
                MESH_TUNNELS[node_id] = st
                t.start()


def mesh_tunnel_stop_all():
    # Bridge shutdown / Mesh off: every tunnel child is
    # terminated cleanly — no orphan ssh is left holding a
    # forward for a bridge that is gone.
    with MESH_TUNNEL_LOCK:
        entries = list(MESH_TUNNELS.values())
        MESH_TUNNELS.clear()
        for st in entries:
            st["stop"] = True
            proc = st.get("proc")
            if proc is not None:
                try:
                    proc.terminate()
                except Exception:
                    pass


def mesh_start_listener():
    global MESH_SERVERS
    # Build 7 (#87): the peer health thread starts with the Mesh
    # door — at boot and at enable alike (this is the one place
    # both paths meet). It idles while Mesh is off or unpaired.
    mesh_ensure_health_thread()
    if MESH_SERVERS:
        return
    cfg = load_config()
    ms = mesh_settings(cfg)
    if not ms["enabled"]:
        return
    ok, _err = mesh_ensure_cert()
    if not ok:
        return
    cert_path, key_path = mesh_active_cert()
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_cert_chain(cert_path, key_path)
    servers = []
    for door_port in ms["listenPorts"]:
        try:
            server = _MeshDoorServer(("0.0.0.0", door_port), MeshHandler)
            server.mesh_tls_context = ctx
            # The accept side of get_request waits on a short
            # fuse: its loop skips dropped and failed knocks
            # without returning, and a bare blocking accept
            # would park the serve thread past shutdown's
            # reach. A timeout raise from accept is an OSError,
            # which socketserver's request loop already absorbs
            # — the serve loop simply comes back around and
            # checks its shutdown flag. Accepted sockets are
            # re-timed by get_request itself before any use.
            server.socket.settimeout(0.5)
            t = threading.Thread(target=server.serve_forever, daemon=True)
            t.start()
            servers.append(server)
        except Exception:
            # One number being taken (something else already
            # listens there) never closes the whole door: the
            # remaining ports still serve, and the miss is named.
            print("Navi Mesh: door port " + str(door_port) +
                  " could not be bound — the door serves on its "
                  "remaining ports.", flush=True)
    MESH_SERVERS = servers
    # Build 4 (#178): the managed backup tunnels reconcile with
    # the same start that opens the door.
    mesh_tunnel_sync()


def mesh_stop_listener():
    global MESH_SERVERS
    mesh_tunnel_stop_all()
    for s in MESH_SERVERS:
        try:
            s.shutdown()
        except Exception:
            pass
        try:
            s.server_close()
        except Exception:
            pass
    MESH_SERVERS = []

def read_file(path):
    try:
        with open(path) as f:
            return f.read()
    except Exception:
        return ""


def app_version_status(cfg=None):
    # The VERSION file's optional build status: the number may be
    # followed by a status word - "0.1.8 edge" - or carry it as a
    # suffix - "0.1.8-edge". Edge and beta builds wear a
    # single-letter tag in the menu footer (e / b); a full release
    # wears none. Anything unrecognized reads as stable. When the file is
    # missing entirely, the status of the release this bridge
    # shipped in answers — never the preserved config's record,
    # which belongs to an older build (see SHIPPED_VERSION above).
    text = read_file(VERSION_PATH).strip()
    if not text:
        return SHIPPED_VERSION_STATUS
    raw = text.splitlines()[0].strip().lower()
    # The pre-release word is "edge" (build .83); "alpha" is the
    # retired word from before the sync and still reads as edge,
    # so a VERSION file written by an older build is never
    # misread as a full release.
    for word in ("edge", "beta", "alpha"):
        if raw.endswith(" " + word) or raw.endswith("-" + word):
            return "edge" if word == "alpha" else word
    return "stable"


def app_version(cfg=None):
    # The VERSION file is the source of truth (see VERSION_PATH).
    # The reported number never carries the status word — the
    # status rides separately as versionStatus, and the dashboard
    # renders the two together ("v0.1.8e" on an Edge build). When
    # the file is
    # missing, the release this bridge shipped in answers. The
    # config's recorded version is NOT a fallback: the config is
    # preserved across installs, so its version is whichever build
    # installed first (the retired 1.0.0 placeholder, on machines
    # installed before the release prep) — trusting it is what made
    # the dashboard's footer flash the baked 0.1.0 and then revert.
    def _number(raw):
        raw = raw.strip()
        if not raw:
            return ""
        token = raw.split()[0]
        for word in ("edge", "alpha", "beta", "stable"):
            if token.lower().endswith("-" + word):
                token = token[: -(len(word) + 1)]
                break
        return token
    text = read_file(VERSION_PATH).strip()
    if text:
        n = _number(text.splitlines()[0])
        if n:
            return n
    return SHIPPED_VERSION


def persona_prompt(cfg, operator_name="", memory_mode="autonomous"):
    parts = [read_file(SOUL_PATH).strip(), read_file(MEMORY_PATH).strip()]
    parts = [p for p in parts if p]
    if not parts:
        parts = [DEFAULT_PERSONA]
    prompt = "\n\n".join(parts)
    # The UI names the operator by the name selected in Settings;
    # until one is selected, the default form of address is
    # "Operator" (the old Hello_Name() placeholder is retired).
    name = (operator_name or cfg.get("operator_name") or "").strip() or "Operator"
    prompt += (
        "\n\nAddress your operator as " + name + " — that is the name "
        "they have asked you to use. Use it naturally, the way someone "
        "who knows them would."
    )
    # Memory from chat (build .81, issue #72): she may pull memories
    # out of the conversation on her own judgment, inside the same
    # reply she is already generating — no second inference pass, so
    # modest hardware pays nothing extra. The bridge strips the
    # marker lines before the reply is shown or spoken and returns
    # the saves as data; the dashboard files them and shows the
    # operator a receipt of exactly what was saved, with an Undo.
    # In trigger-only mode the instruction is not sent at all, and
    # the bridge drops any marker a reply still carries.
    if memory_mode != "trigger":
        prompt += (
            "\n\nMemory: you keep long-term memory entries for your operator. "
            "When this conversation produces something durable — a fact about "
            "them, a preference, a decision, a commitment, or a correction to "
            "something you already carry — you may save it by ending your reply "
            "with a line of exactly this form: [[remember: <the fact, stated "
            "plainly in one sentence>]]. At most two per reply, and only when "
            "it truly matters; small talk and command output are not memories. "
            "If the operator corrects a fact, save the corrected fact the same "
            "way — it replaces the old entry rather than piling up beside it. "
            "Never save secrets (passwords, keys, card numbers, tokens): a "
            "save like that is refused before it is written. The marker line "
            "is removed before the operator sees your reply, and they are "
            "shown a receipt of exactly what you saved — never mention the "
            "marker itself."
        )
    return prompt


# ---------------------------------------------------------------------
# Memory from chat (build .81, issue #72).
#
# Two doors, one store. The trigger — "remember this", "remember:
# …", "save this to your memory" — is deterministic bridge work,
# decided before the model route like DHD intents and Vault
# filing: the fact is extracted here and carried back in the
# payload, and the dashboard files it and shows the receipt. Her
# own judgment rides the model's reply: the persona prompt teaches
# her the [[remember: …]] marker, extract_memory_markers strips
# those lines from the reply before it is shown or spoken, and the
# saves travel back as payload data — never as words in her text.
# The entries themselves live with the dashboard (its local store,
# like the rest of her remembered facts); corrections are matched
# there by the same key/overlap rule memory_fact_key mirrors here.
# Secrets are screened on both doors and refused before anything
# is written, and no save is ever silent: every outcome rides the
# payload back to a receipt in the thread.

MEMORY_SECRET_RE = re.compile(r"\b(password|passcode|passwd|api[\s_-]?key|secret|token|credential|ssn|social security|credit card|card number|cvv)\b", re.I)
MEMORY_LONG_DIGITS_RE = re.compile(r"\b(?:\d[ -]?){13,19}\b")
MEMORY_MARKER_RE = re.compile(r"^\s*\[\[remember:\s*(.+?)\s*\]\]\s*$", re.I | re.M)


def memory_looks_secret(text):
    return bool(MEMORY_SECRET_RE.search(text or "") or MEMORY_LONG_DIGITS_RE.search(text or ""))


def normalize_memory_fact(text):
    t = re.sub(r"\s+", " ", (text or "").strip()).strip("\"'")
    if not t:
        return ""
    t = t[0].upper() + t[1:]
    if not re.search(r"[.!?]$", t):
        t += "."
    return t


def memory_fact_key(text):
    t = (text or "").lower()
    if re.search(r"(?:operator'?s|my) name is\b", t) or re.match(r"name\s*:", t):
        return "name"
    if re.search(r"(?:operator lives|i live|i'?m based|operator'?s (?:home|hometown) is|my (?:home|hometown) is)\b", t):
        return "home"
    if re.search(r"(?:operator works|i work|my job is|operator'?s job is)\b", t):
        return "work"
    m = re.search(r"favou?rite\s+([a-z][a-z -]*?)\s+is\b", t)
    if m:
        return "favorite:" + m.group(1).strip()
    return None


def memory_trigger(text):
    # The operator's trigger. Returns ("fact", payload),
    # ("prior", None) when the ask points at the previous message
    # ("remember this" with nothing after it), or None when the
    # line is not a trigger — questions about remembering ("do you
    # remember…", "remember when…") are talk, not triggers.
    line = (text or "").strip()
    if line.startswith(">"):
        line = line[1:].strip()
    if "?" in line and re.match(r"(?i)(do|did|can|could|what|why|how|remember when)\b", line):
        return None
    m = re.match(r"(?i)(?:please\s+)?(?:navi[,\s]+)?(?:save|write|put)\s+(?:this|that)\s+(?:to|in|into)\s+(?:your\s+)?memory\s*[:\-]?\s*(.*)$", line)
    if m:
        rest = (m.group(1) or "").strip()
        return ("fact", rest) if rest else ("prior", None)
    m = re.match(r"(?i)(?:please\s+)?(?:navi[,\s]+)?remember[\s:]+(.*)$", line)
    if not m:
        return None
    rest = (m.group(1) or "").strip()
    if not rest or re.match(r"(?i)(when|me|us|the time)\b", rest):
        return None
    pm = re.match(r"(?i)(?:this|that)\b\s*[:\-—,]?\s*(.*)$", rest)
    if pm:
        rest = (pm.group(1) or "").strip()
        return ("fact", rest) if rest else ("prior", None)
    if rest[:1] in (":", "-"):
        rest = rest[1:].strip()
        return ("fact", rest) if rest else ("prior", None)
    return ("fact", rest)


def extract_memory_markers(reply, memory_mode):
    # Strip [[remember: …]] lines from a model reply. Returns
    # (clean_reply, saves, refused_count). Markers are honored
    # only when the mode lets her save on her own judgment; in
    # trigger mode they are stripped and dropped. Secret-shaped
    # saves count as refused — never written, and the dashboard's
    # receipt says so.
    saves = []
    refused = [0]

    def _take(m):
        fact = normalize_memory_fact(m.group(1))
        if not fact:
            return ""
        if memory_mode == "trigger":
            return ""
        if memory_looks_secret(fact):
            refused[0] += 1
            return ""
        saves.append({"text": fact, "origin": "autonomous", "key": memory_fact_key(fact), "proposed": memory_mode == "ask"})
        return ""

    clean = MEMORY_MARKER_RE.sub(_take, reply or "")
    clean = re.sub(r"\n{3,}", "\n\n", clean).strip()
    return clean, saves[:2], refused[0]


LIST_INTENT = re.compile(
    r"(?:show|list|display|contents of|what's in|what is in)[:,]?\s+(?:me\s+)?(?:the\s+)?(?:contents\s+of\s*:?\s*)?(/[^\s\"']+|~[^\s\"']*)",
    re.IGNORECASE,
)


def expand(path):
    return os.path.expanduser(path)


# ---------------------------------------------------------------------
# Sudo mode (v0.3.0e — Build 3, #209 leg A2): sudo's own timestamp,
# no stored password, every elevated Navi run on the record.
#
# The model: the operator arms sudo mode in Settings. Arming feeds
# the operator's password ONCE to "sudo -v" on stdin — the password
# transits this process's memory for that single call and is never
# stored, logged, synced or written anywhere. What persists is one
# boolean in navi.config.json (sudo.enabled) plus sudo's own
# credential timestamp. While the timestamp lives, elevated
# commands run as "sudo -n <cmd>" (never prompt); when it lapses
# the state reads EXPIRED and elevated commands refuse honestly
# until the operator re-warms. Disarming flips the boolean and
# kills the timestamp (sudo -k) — instant and total.
#
# Why this replaced the persistent root shell: the shell WAS the
# credential, so the bridge held root indefinitely with no expiry
# semantics at all. The timestamp model borrows sudo's own expiry
# instead. And every elevated command Navi runs lands on the
# record: config/sudo-audit.jsonl, append-only, surfaced read-only
# in Settings, deliberately NOT part of the Sync set.
#
# Session note: sudo timestamps key by session id (and tty, where
# configured). Every child the bridge spawns without setsid shares
# the bridge's session, so one child's "sudo -v" warms the next
# child's "sudo -n" — the sudo1 bench proves this on the shipped
# package. Interactive shell sessions (CLI Mode, below) run under
# their own session id and deliberately do NOT inherit the warmth:
# the operator's own sudo inside their terminal prompts
# in-terminal, as it should, and Navi never drives elevation
# through the PTY at all.

SUDO_NOISE_LINES = (
    "verification timed out",
    "verification failed",
    "failed to verify fingerprint",
    "could not verify fingerprint",
)

SUDO_AUDIT_PATH = os.path.join(
    os.path.dirname(CONFIG_PATH), "sudo-audit.jsonl")
_sudo_lock = threading.Lock()
_sudo_probe_cache = {"at": 0.0, "warm": False}


def _is_sudo_noise(line):
    low = line.strip().lower()
    if not low:
        return False
    if low in SUDO_NOISE_LINES:
        return True
    # pam_fprintd's prompt names whichever finger is enrolled:
    # "Place your right index finger on the fingerprint reader".
    return low.startswith("place your") and "fingerprint reader" in low


def clean_sudo_output(text):
    if not text:
        return text
    kept = [ln for ln in text.split("\n") if not _is_sudo_noise(ln)]
    return "\n".join(kept).strip()


def sudo_settings(cfg=None):
    cfg = cfg if cfg is not None else load_config()
    raw = cfg.get("sudo") if isinstance(cfg, dict) else None
    return {"enabled": bool(isinstance(raw, dict) and raw.get("enabled"))}


def _sudo_probe():
    # Is sudo's credential timestamp warm for the bridge's session
    # right now? "sudo -n true" answers without ever prompting.
    # Cached for a few seconds: the status column polls.
    now = time.time()
    if now - _sudo_probe_cache["at"] < 5:
        return _sudo_probe_cache["warm"]
    warm = False
    try:
        proc = subprocess.run(
            ["sudo", "-n", "true"], capture_output=True, text=True,
            timeout=10, cwd=shell_cwd())
        warm = proc.returncode == 0
    except Exception:
        warm = False
    _sudo_probe_cache["at"] = now
    _sudo_probe_cache["warm"] = warm
    return warm


def sudo_state(cfg=None):
    # The one status shape every surface reads: the Settings card,
    # the status column, /api/status. "off" = the feature is not
    # enabled; "armed" = enabled and the timestamp is warm;
    # "expired" = enabled but the timestamp has lapsed.
    if not sudo_settings(cfg)["enabled"]:
        return {"enabled": False, "state": "off"}
    return {"enabled": True,
            "state": "armed" if _sudo_probe() else "expired"}


def sudo_arm(password):
    # Warm sudo's timestamp with the operator's password, fed once
    # on stdin and forgotten — never stored, logged or synced. On
    # success the feature is enabled in config; on failure nothing
    # is enabled and the refusal carries sudo's own words (cleaned
    # of PAM chatter). Returns (ok, error_or_None). The 120s budget
    # exists because the one PAM conversation may have to wait out
    # a fingerprint reader before the password is even tried.
    password = (password or "").split("\n")[0]
    if not password:
        return False, "Enter the sudo password to arm sudo mode."
    try:
        proc = subprocess.run(
            "sudo -S -p '' -v", shell=True, capture_output=True,
            text=True, timeout=120, cwd=shell_cwd(),
            input=password + "\n")
    except Exception as e:
        return False, "sudo could not be run: " + str(e)
    _sudo_probe_cache["at"] = 0.0
    if proc.returncode != 0 or not _sudo_probe():
        err_text = clean_sudo_output(
            ((proc.stderr or "") + "\n" + (proc.stdout or "")).strip())
        lines = [ln for ln in err_text.split("\n")
                 if "try again" not in ln.lower()]
        msg = lines[-1].strip() if lines else "sudo rejected that password."
        return False, msg
    cfg = load_config()
    cfg["sudo"] = {"enabled": True}
    save_config(cfg)
    return True, None


def sudo_disarm():
    # Instant and total: the boolean flips AND the timestamp dies,
    # so nothing the bridge runs can elevate again until a fresh
    # arm — not even inside sudo's own timeout window.
    cfg = load_config()
    cfg["sudo"] = {"enabled": False}
    save_config(cfg)
    try:
        subprocess.run(["sudo", "-k"], capture_output=True, timeout=10,
                       cwd=shell_cwd())
    except Exception:
        pass
    _sudo_probe_cache["at"] = 0.0
    _sudo_probe_cache["warm"] = False
    return sudo_state(cfg)


def sudo_audit_append(command, chat_id, exit_code):
    # Append-only. One JSON object per elevated run: when, the
    # command as run, the chat it served, how it ended. Returns
    # False when the record could not be written — the caller
    # surfaces that on the result rather than hiding it.
    entry = {"ts": int(time.time()), "command": command,
             "chat": chat_id, "exit": exit_code}
    try:
        os.makedirs(os.path.dirname(SUDO_AUDIT_PATH), exist_ok=True)
        with open(SUDO_AUDIT_PATH, "a", encoding="utf-8") as fh:
            fh.write(json.dumps(entry) + "\n")
        return True
    except Exception:
        return False


def sudo_audit_entries(limit=100):
    try:
        with open(SUDO_AUDIT_PATH, encoding="utf-8") as fh:
            lines = fh.read().splitlines()
    except Exception:
        return []
    out = []
    for ln in lines[-max(1, int(limit)):]:
        try:
            out.append(json.loads(ln))
        except Exception:
            continue
    return out


def path_complete(prefix, base=None):
    """One-level completion for the chat composer (#27d). Pure helper:
    lists only the prefix's parent directory, directories first then
    files (alphabetical), at most 12 entries, dotfiles only when the
    partial name starts with a dot. Replacement text preserves the
    operator's prefix style (~ stays ~, relative stays relative); a
    directory's replacement ends with '/'."""
    prefix = prefix or ""
    slash = prefix.rfind("/")
    dirpart = prefix[:slash + 1] if slash != -1 else ""
    partial = prefix[slash + 1:] if slash != -1 else prefix
    if prefix.startswith("~"):
        expanded = os.path.expanduser(dirpart if dirpart else "~")
        listdir = expanded
    elif dirpart.startswith("/"):
        listdir = dirpart
    else:
        root = base if base is not None else shell_cwd()
        listdir = os.path.join(root, dirpart) if dirpart else root
    try:
        names = os.listdir(listdir)
    except Exception:
        return []
    dirs, files = [], []
    for nm in names:
        if nm.startswith(".") and not partial.startswith("."):
            continue
        if not nm.lower().startswith(partial.lower()):
            continue
        full = os.path.join(listdir, nm)
        try:
            is_dir = os.path.isdir(full)
        except Exception:
            is_dir = False
        (dirs if is_dir else files).append(nm)
    out = []
    for nm in sorted(dirs):
        out.append({"name": nm, "description": "Folder", "replacement": dirpart + nm + "/", "isDir": True})
    for nm in sorted(files):
        out.append({"name": nm, "description": "File", "replacement": dirpart + nm, "isDir": False})
    return out[:12]


def run_shell(command, timeout=20, sudo=False, sudo_password="",
              chat_id=None, audit=False):
    raw = (command or "").strip()
    body = raw
    wants_sudo = bool(sudo)
    if body == "sudo" or body.startswith("sudo "):
        # A literal "sudo " prefix escalates this one command either way.
        wants_sudo = True
        body = body[4:].lstrip()
    # What the transcript shows — never the password, never the
    # plumbing behind it.
    display = ("sudo " + body) if wants_sudo and body else body
    if wants_sudo and body:
        password = (sudo_password or "").split("\n")[0]
        if password:
            # Explicit-password path — internal callers only (the
            # DHD prerequisite installer, whose password the
            # operator types into that flow for that purpose). One
            # sudo process, the password on stdin, never stored,
            # logged or synced.
            proc = subprocess.run(
                "sudo -S -p '' " + body, shell=True,
                capture_output=True, text=True, timeout=timeout,
                cwd=shell_cwd(), input=password + "\n",
            )
            output = clean_sudo_output(
                (proc.stdout + proc.stderr).strip())
            if proc.returncode != 0 and (
                "password" in output.lower()
                or "sorry" in output.lower()
            ):
                output += (
                    "\n(sudo rejected the password given to that "
                    "installer step.)"
                )
            return {
                "command": display,
                "output": output,
                "exit_code": proc.returncode,
            }
        # Sudo mode proper (Build 3): the timestamp model. Nothing
        # here ever prompts; a command that cannot run says why.
        st = sudo_state()
        if not st["enabled"]:
            return {
                "command": display,
                "output": "(sudo mode is off — this command was not "
                          "run. Arm sudo mode in Settings → System "
                          "if it should run elevated.)",
                "exit_code": 1,
            }
        if st["state"] != "armed":
            return {
                "command": display,
                "output": "(sudo mode is on but its timestamp has "
                          "expired — this command was not run. "
                          "Re-warm sudo mode in Settings → System.)",
                "exit_code": 1,
            }
        proc = subprocess.run(
            "sudo -n " + body, shell=True, capture_output=True,
            text=True, timeout=timeout, cwd=shell_cwd(),
        )
        output = clean_sudo_output((proc.stdout + proc.stderr).strip())
        if proc.returncode != 0 and "password is required" in output:
            # The timestamp died between the probe and the run.
            _sudo_probe_cache["at"] = 0.0
            _sudo_probe_cache["warm"] = False
            output += (
                "\n(sudo mode's timestamp expired before this "
                "command ran — re-warm sudo mode in Settings → "
                "System.)"
            )
        result = {
            "command": display,
            "output": output,
            "exit_code": proc.returncode,
        }
        if audit and not sudo_audit_append(display, chat_id,
                                           proc.returncode):
            result["output"] = (output +
                "\n(warning: the sudo audit log could not be "
                "written — this elevated run is not on the "
                "record.)").strip()
        return result
    proc = subprocess.run(
        body, shell=True, capture_output=True, text=True,
        timeout=timeout, cwd=shell_cwd(),
    )
    return {
        "command": display,
        "output": (proc.stdout + proc.stderr).strip(),
        "exit_code": proc.returncode,
    }


# ---------------------------------------------------------------------
# CLI Mode: interactive shell sessions (v0.3.0e — Build 3, #209
# leg A1). One standing PTY-backed session per chat.
#
# A session is born on first interactive need — the operator opens
# it from the composer, or Navi drives one into existence with a
# [[shell]] block — and then survives messages and window reloads
# (it lives here, in the bridge, not in the page). It ends when
# someone closes it, when its shell exits, after 30 minutes
# without input (the sweeper says so in the thread), or when the
# bridge dies — and a bridge death is never papered over: the
# events stay queryable as a tombstone until the bridge restarts,
# and the persisted thread record shows a session with no closing
# note, which the page names as a bridge restart.
#
# Everything a session does is an event on the chat's record:
# inputs badged by author ("operator" / "navi"), output as it
# streams. A program that takes the alternate screen is held on a
# real grid (below) for the desktop's terminal pane; its output
# does not pollute the streamed record, and entering/leaving the
# alternate screen is itself noted in the thread.
#
# Elevation never rides the PTY: the session runs under its own
# session id, so sudo's timestamp warmth from sudo mode does not
# leak into it (and Navi has no password to type here anyway).
# The operator's own sudo inside their session prompts in-terminal.

import codecs
import fcntl
import pty
import signal
import termios

SHELL_IDLE_SECONDS = 1800  # 30 minutes without input

_ANSI_SEQ = re.compile(
    r"\x1b(?:\[[0-9;?]*[ -/]*[@-~]|\][^\x07\x1b]*(?:\x07|\x1b\\)|.)",
    re.S)


def _strip_ansi(text):
    if not text:
        return ""
    return _ANSI_SEQ.sub("", text).replace("\r", "")


class _TermGrid:
    """A minimal VT grid — enough of xterm to hold a full-screen
    program's frame for the desktop pane: printable text with
    wrap, CR/LF/BS, cursor addressing, erases, insert/delete
    lines and chars, scrolling, and the alternate screen. SGR
    and friends are parsed and ignored. It is a faithful record
    of the frame, not a terminal emulator competing with xterm."""

    def __init__(self, cols=80, rows=24):
        self.cols = max(20, int(cols or 80))
        self.rows = max(5, int(rows or 24))
        self.main = self._blank()
        self.alt = None
        self.cx = 0
        self.cy = 0
        self.in_alt = False

    def _blank(self):
        return [[" "] * self.cols for _ in range(self.rows)]

    @property
    def grid(self):
        return self.alt if self.in_alt else self.main

    def lines(self):
        return ["".join(r).rstrip() for r in self.grid]

    def resize(self, cols, rows):
        cols = max(20, int(cols or self.cols))
        rows = max(5, int(rows or self.rows))
        for name in ("main", "alt"):
            g = getattr(self, name)
            if g is None:
                continue
            ng = [[" "] * cols for _ in range(rows)]
            for y in range(min(rows, len(g))):
                for x in range(min(cols, len(g[y]))):
                    ng[y][x] = g[y][x]
            setattr(self, name, ng)
        self.cols, self.rows = cols, rows
        self.cx = min(self.cx, cols - 1)
        self.cy = min(self.cy, rows - 1)

    def _scroll_up(self, n=1):
        g = self.grid
        for _ in range(n):
            g.pop(0)
            g.append([" "] * self.cols)

    def _scroll_down(self, n=1):
        g = self.grid
        for _ in range(n):
            g.pop()
            g.insert(0, [" "] * self.cols)

    def _newline(self):
        self.cx = 0
        self.cy += 1
        if self.cy >= self.rows:
            self._scroll_up(1)
            self.cy = self.rows - 1

    def _put(self, ch):
        self.grid[self.cy][self.cx] = ch
        self.cx += 1
        if self.cx >= self.cols:
            self._newline()

    def feed(self, text):
        i = 0
        n = len(text)
        while i < n:
            ch = text[i]
            if ch == "\x1b":
                m = re.match(r"\x1b\[([0-9;?]*)([@-~])", text[i:])
                if m:
                    self._csi(m.group(1), m.group(2))
                    i += m.end()
                    continue
                m = re.match(r"\x1b\][^\x07\x1b]*(?:\x07|\x1b\\)",
                             text[i:])
                if m:
                    i += m.end()
                    continue
                i += 2 if i + 1 < n else 1
                continue
            if ch == "\n":
                self._newline()
            elif ch == "\r":
                self.cx = 0
            elif ch == "\b":
                self.cx = max(0, self.cx - 1)
            elif ch == "\t":
                self.cx = min(self.cols - 1, (self.cx // 8 + 1) * 8)
            elif ch >= " ":
                self._put(ch)
            i += 1

    def _csi(self, params, final):
        private = params.startswith("?")
        nums = [p for p in params.lstrip("?").split(";") if p != ""]
        def num(k, default):
            try:
                v = int(nums[k])
                return v if v != 0 else default
            except Exception:
                return default
        g = self.grid
        if private:
            modes = [int(p) for p in nums if p.isdigit()]
            if any(m in (1049, 1047, 47) for m in modes):
                if final == "h" and not self.in_alt:
                    self.alt = self._blank()
                    self.in_alt = True
                elif final == "l" and self.in_alt:
                    self.in_alt = False
                    self.alt = None
            return
        if final in ("H", "f"):
            self.cy = max(0, min(self.rows - 1, num(0, 1) - 1))
            self.cx = max(0, min(self.cols - 1, num(1, 1) - 1))
        elif final == "A":
            self.cy = max(0, self.cy - num(0, 1))
        elif final == "B":
            self.cy = min(self.rows - 1, self.cy + num(0, 1))
        elif final == "C":
            self.cx = min(self.cols - 1, self.cx + num(0, 1))
        elif final == "D":
            self.cx = max(0, self.cx - num(0, 1))
        elif final == "G":
            self.cx = max(0, min(self.cols - 1, num(0, 1) - 1))
        elif final == "d":
            self.cy = max(0, min(self.rows - 1, num(0, 1) - 1))
        elif final == "J":
            mode = num(0, 0)
            if mode == 0:
                for x in range(self.cx, self.cols):
                    g[self.cy][x] = " "
                for y in range(self.cy + 1, self.rows):
                    g[y] = [" "] * self.cols
            elif mode == 1:
                for x in range(0, self.cx + 1):
                    g[self.cy][x] = " "
                for y in range(0, self.cy):
                    g[y] = [" "] * self.cols
            else:
                for y in range(self.rows):
                    g[y] = [" "] * self.cols
        elif final == "K":
            mode = num(0, 0)
            if mode == 0:
                rng = range(self.cx, self.cols)
            elif mode == 1:
                rng = range(0, self.cx + 1)
            else:
                rng = range(0, self.cols)
            for x in rng:
                g[self.cy][x] = " "
        elif final == "L":
            for _ in range(num(0, 1)):
                g.insert(self.cy, [" "] * self.cols)
                g.pop()
        elif final == "M":
            for _ in range(num(0, 1)):
                g.pop(self.cy)
                g.append([" "] * self.cols)
        elif final == "P":
            row = g[self.cy]
            for _ in range(num(0, 1)):
                if self.cx < len(row):
                    row.pop(self.cx)
                    row.append(" ")
        elif final == "@":
            row = g[self.cy]
            for _ in range(num(0, 1)):
                row.insert(self.cx, " ")
                row.pop()
        elif final == "S":
            self._scroll_up(num(0, 1))
        elif final == "T":
            self._scroll_down(num(0, 1))
        # m (SGR), r, s, u, and everything else: parsed, ignored.


def _key_display(text):
    # How a raw key burst reads in the thread record: printable
    # runs kept, control characters named.
    out = []
    for ch in text:
        if ch in ("\r", "\n"):
            out.append("⏎")
        elif ch == "\x03":
            out.append("^C")
        elif ch == "\x04":
            out.append("^D")
        elif ch == "\x1a":
            out.append("^Z")
        elif ch == "\x09":
            out.append("⇥")
        elif ch < " ":
            out.append("^" + chr(ord(ch) + 64))
        else:
            out.append(ch)
    return "".join(out)


class ShellSession:
    """One chat's standing terminal. See the section header."""

    def __init__(self, chat_id, cols=80, rows=24):
        self.chat_id = str(chat_id)
        self.grid = _TermGrid(cols, rows)
        self.events = []
        self.seq = 0
        self.lock = threading.Lock()
        self.created_at = time.time()
        self.last_input_at = time.time()
        self.alive = True
        self.close_reason = None
        self._pending = ""
        self._pending_at = 0.0
        self._tail = ""
        self._was_alt = False
        self._decoder = codecs.getincrementaldecoder("utf-8")("replace")
        master, slave = pty.openpty()
        try:
            fcntl.ioctl(slave, termios.TIOCSWINSZ,
                        struct.pack("HHHH", self.grid.rows,
                                    self.grid.cols, 0, 0))
        except Exception:
            pass
        env = dict(os.environ)
        env["TERM"] = "xterm-256color"

        def _preexec():
            os.setsid()
            try:
                fcntl.ioctl(slave, termios.TIOCSCTTY, 0)
            except Exception:
                pass

        self.proc = subprocess.Popen(
            ["/bin/bash", "-i"], stdin=slave, stdout=slave,
            stderr=slave, cwd=shell_cwd(), env=env,
            preexec_fn=_preexec, close_fds=True)
        os.close(slave)
        self.master = master
        os.set_blocking(master, False)
        self._emit("note", None, "shell session opened")
        self._reader = threading.Thread(target=self._read_loop,
                                        daemon=True)
        self._reader.start()

    # -- events ---------------------------------------------------
    def _emit(self, kind, author, text):
        with self.lock:
            self.seq += 1
            self.events.append({
                "seq": self.seq, "kind": kind, "author": author,
                "text": text, "ts": int(time.time())})
            if len(self.events) > 400:
                del self.events[:len(self.events) - 400]

    def events_since(self, since):
        with self.lock:
            return [dict(e) for e in self.events if e["seq"] > since]

    def output_tail(self, limit=1200):
        with self.lock:
            return self._tail[-limit:]

    def wait_settled(self, quiet=0.5, cap=4.0, floor=0.9):
        # A newborn session's startup output — the shell's
        # banner, the distro's welcome mat — lands over the
        # first moments of its life, while an input typed at
        # once is emitted into the record immediately and the
        # banner flushes in BEHIND it: the field shape that
        # buried an uptime answer under thirty lines of Bazzite
        # MOTD (v0.4.2e, #232). Wait until the stream has been
        # quiet for `quiet` seconds — no new events, nothing
        # waiting to flush — at most `cap`, so the banner takes
        # its place at the session's opening, before any driven
        # line. The `floor` is the session's minimum age before
        # quiet counts at all: a banner still on its way (a
        # login script that sleeps before it prints) is silent
        # too, and silence that young proves nothing. Returns
        # True when the stream settled, False on the cap (the
        # caller types anyway; a slow shell is still a shell).
        started = time.time()
        deadline = started + cap
        last_seq = -1
        quiet_since = None
        while time.time() < deadline:
            with self.lock:
                seq = self.seq
                pending = bool(self._pending)
            if seq == last_seq and not pending:
                if quiet_since is None:
                    quiet_since = time.time()
                elif time.time() - quiet_since >= quiet \
                        and time.time() - started >= floor:
                    return True
            else:
                last_seq = seq
                quiet_since = None
            time.sleep(0.05)
        return False

    # -- output path ------------------------------------------------
    def _on_data(self, text):
        if not text:
            return
        with self.lock:
            self.grid.feed(text)
            alt = self.grid.in_alt
        if alt != self._was_alt:
            self._was_alt = alt
            if alt:
                self._flush_pending(force=True)
                self._emit("note", None,
                           "a program took the full screen — on the "
                           "desktop it renders in the terminal pane")
            else:
                self._emit("note", None,
                           "the full-screen program exited — back "
                           "to the streamed record")
        if not alt:
            self._pending += text
            if not self._pending_at:
                self._pending_at = time.time()
            self._flush_pending()

    def _flush_pending(self, force=False):
        if not self._pending:
            return
        if not force and len(self._pending) < 8192 \
                and time.time() - self._pending_at < 0.15:
            return
        raw, self._pending = self._pending, ""
        self._pending_at = 0.0
        text = _strip_ansi(raw)
        if text.strip():
            with self.lock:
                self._tail = (self._tail + text)[-8000:]
            self._emit("out", None, text.rstrip("\n"))

    def _read_loop(self):
        while True:
            try:
                r, _, _ = select.select([self.master], [], [], 0.2)
            except Exception:
                break
            if r:
                try:
                    chunk = os.read(self.master, 65536)
                except OSError:
                    break
                if not chunk:
                    break
                self._on_data(self._decoder.decode(chunk))
            self._flush_pending()
            if self.proc.poll() is not None:
                try:
                    while True:
                        chunk = os.read(self.master, 65536)
                        if not chunk:
                            break
                        self._on_data(self._decoder.decode(chunk))
                except Exception:
                    pass
                break
        self._flush_pending(force=True)
        code = self.proc.poll()
        with self.lock:
            was = self.alive
            self.alive = False
        if was:
            self._emit("note", None,
                       "the shell exited" +
                       ("" if code is None else
                        " (exit " + str(code) + ")") +
                       " — the session is closed")
            _shell_session_died(self)

    # -- input path ---------------------------------------------------
    def input(self, text, author, raw=False):
        if not self.alive:
            return False
        try:
            os.write(self.master, text.encode("utf-8", "replace"))
        except OSError:
            return False
        self.last_input_at = time.time()
        shown = _key_display(text) if raw else text.rstrip("\r\n")
        if shown or not raw:
            self._emit("in", author, shown)
        return True

    def resize(self, cols, rows):
        with self.lock:
            self.grid.resize(cols, rows)
            rows_now, cols_now = self.grid.rows, self.grid.cols
        try:
            fcntl.ioctl(self.master, termios.TIOCSWINSZ,
                        struct.pack("HHHH", rows_now, cols_now, 0, 0))
        except Exception:
            pass

    def kill(self, reason):
        with self.lock:
            was = self.alive
            self.alive = False
            self.close_reason = reason
        if was:
            self._flush_pending(force=True)
            self._emit("note", None, "session closed (" + reason + ")")
        try:
            os.killpg(self.proc.pid, signal.SIGTERM)
        except Exception:
            pass
        try:
            self.proc.wait(timeout=2)
        except Exception:
            try:
                os.killpg(self.proc.pid, signal.SIGKILL)
            except Exception:
                pass
        try:
            os.close(self.master)
        except Exception:
            pass


_shell_sessions = {}
_shell_tombstones = {}
_shell_mgr_lock = threading.Lock()
_shell_sweeper_on = [False]


def _shell_sweeper():
    while True:
        time.sleep(20)
        now = time.time()
        for chat_id, sess in list(_shell_sessions.items()):
            if sess.alive and \
                    now - sess.last_input_at > SHELL_IDLE_SECONDS:
                shell_session_close(chat_id, "idle for 30 minutes")


def _ensure_sweeper():
    if not _shell_sweeper_on[0]:
        _shell_sweeper_on[0] = True
        threading.Thread(target=_shell_sweeper, daemon=True).start()


def shell_session_open(chat_id, cols=80, rows=24):
    key = str(chat_id)
    with _shell_mgr_lock:
        sess = _shell_sessions.get(key)
        if sess is not None and sess.alive:
            return sess
        sess = ShellSession(key, cols, rows)
        _shell_sessions[key] = sess
        _shell_tombstones.pop(key, None)
        _ensure_sweeper()
        return sess


def shell_session_get(chat_id):
    sess = _shell_sessions.get(str(chat_id))
    return sess if (sess is not None and sess.alive) else None


def shell_session_input(chat_id, text, author, raw=False, create=False):
    sess = shell_session_get(chat_id)
    if sess is None and create:
        sess = shell_session_open(chat_id)
    if sess is None:
        return False
    return sess.input(text, author, raw=raw)


def shell_session_close(chat_id, reason="closed by the operator"):
    key = str(chat_id)
    with _shell_mgr_lock:
        sess = _shell_sessions.pop(key, None)
        if sess is not None:
            _shell_tombstones[key] = sess
    if sess is not None:
        sess.kill(reason)
    return sess


def _shell_session_died(sess):
    with _shell_mgr_lock:
        if _shell_sessions.get(sess.chat_id) is sess:
            _shell_sessions.pop(sess.chat_id, None)
            _shell_tombstones[sess.chat_id] = sess


def shell_session_state(chat_id, since=0):
    key = str(chat_id)
    sess = _shell_sessions.get(key)
    if sess is not None and sess.alive:
        with sess.lock:
            alt = sess.grid.in_alt
            screen = sess.grid.lines() if alt else None
            cursor = [sess.grid.cx, sess.grid.cy]
            seq = sess.seq
        return {
            "ok": True, "live": True, "ended": False,
            "chatId": key, "altScreen": alt, "screen": screen,
            "cursor": cursor, "seq": seq,
            "events": sess.events_since(since),
            "idleFor": int(time.time() - sess.last_input_at),
        }
    tomb = _shell_tombstones.get(key)
    if tomb is not None:
        return {
            "ok": True, "live": False, "ended": True,
            "chatId": key, "altScreen": False, "screen": None,
            "cursor": None, "seq": tomb.seq,
            "events": tomb.events_since(since),
            "closeReason": tomb.close_reason or "the shell exited",
        }
    return {"ok": True, "live": False, "ended": False,
            "chatId": key, "altScreen": False, "screen": None,
            "cursor": None, "seq": 0, "events": []}


def shell_sessions_kill_all():
    for chat_id in list(_shell_sessions.keys()):
        shell_session_close(chat_id, "the bridge is shutting down")


_SHELL_BLOCK = re.compile(r"\[\[shell\]\]\s*\n?(.*?)\n?\s*\[\[/shell\]\]",
                          re.S | re.I)
_SHELL_BLOCK_OPEN = re.compile(r"\[\[shell\]\]", re.I)

SHELL_TEACHING = (
    "Interactive shell: this chat can hold ONE live terminal "
    "session, and it is a tool in your hand — never your voice. "
    "The shell wakes only when the operator is working the "
    "machine WITH you: he has asked for machine work — files, "
    "processes, services, the state of this box — or the two of "
    "you are already mid-flow in a session and the next move "
    "depends on the terminal's answer. Conversation is answered "
    "in conversation: a question you can answer in words is "
    "answered in words, a remark gets a reply, and a general "
    "question — geography, history, the world — is NEVER a "
    "reason to open the shell, look something up in it, or let "
    "a block stand in for an answer. The bridge enforces this "
    "gate (v0.4.6e, #240): a block in a reply to a conversation "
    "turn is not run at all. When a drive IS warranted, SPEAK "
    "FIRST in the same reply: a sentence in your own words "
    "saying what you are running and why, then end the reply "
    "with the block in its form — an opening line [[shell]], "
    "each line to type exactly as typed, a closing [[/shell]]. "
    "Type commands, never sentences: prose typed into bash is "
    "refused by name before it reaches the terminal. A reply "
    "that is only a block is a failed reply — he asked you, "
    "not the terminal. The lines are typed into the session "
    "badged as yours, its output streams into the thread, and "
    "you will see its tail on your next turn: when that output "
    "answers his question, your next reply says what it showed, "
    "in your own words — the result, never the session's banner "
    "or scaffolding, and never a file or a fact the output did "
    "not actually show. One-shot commands do not need the "
    "session; they run as they always have. Elevation never "
    "rides the session: sudo inside it prompts in-terminal.")

# v0.4.5e (#241) built Route A here: her [[case-intake]] block
# was the trigger and the bridge the acting half. v0.5.14e
# (#289) retired that shape in the field's own words — a plain
# question in the main chat ("What is going on with Apple?")
# was eaten by the intake and answered with a "Case not
# started" refusal, twice. The teaching below is the new law,
# the mirror of the case room's [Working this case] (#286):
# she prints the answer, then offers. Starting is the
# operator's act on the reply's own button; the bridge does
# the creating and the filing; her prose never does.
CASE_TEACHING = (
    "[Starting a case — she prints, then offers (v0.5.14e, #289)] "
    "When a conversation turns up something case-worthy — a threat, "
    "an incident, a thread worth tracking — ANSWER FIRST, in full, "
    "exactly as you would any other question. Nothing about a reply "
    "of yours is parsed for case machinery, and no reply starts a "
    "case, whatever shape it takes. Under every reply you give in a "
    "chat that is not already a case room stands a \"Start a case "
    "from this\" button; pressing it is the operator's act, and it "
    "is the only way a case starts from chat: the bridge composes "
    "the case from that reply exactly as printed — the reply filed "
    "whole as a finding in your name — receipts it, ensures the "
    "case's own chat, and sets the case open. You may suggest "
    "starting a case when the material earns one, and let the "
    "suggestion stand: never wait on the press, never let an "
    "unpressed suggestion hang over the next answer, and never "
    "narrate a case as started that the bridge has not receipted. "
    "The New case button beside the composer is the deterministic "
    "door — it opens a form pre-filled from the conversation for "
    "the operator to confirm or correct before anything exists. "
    "Adding to an EXISTING case is not a start — that is the open "
    "case's own filing, which rides its own door. Filings carry a "
    "DESTINATION (v0.5.15e, #291; completed v0.6.6e, #326/#327): a case has "
    "nine parts — finding, lead, observable, entity, event, "
    "task, timeline, evidence, report — and the button under a "
    "reply names the part it will file to (\"File into this case "
    "— as a lead\"), sensed from the reply's shape: a next-step "
    "list reads as tasks (with its count), a dated narrative as "
    "the timeline, a short typed subject as an entity. When you suggest filing, suggest the part "
    "the content plainly is — a sourced claim is a lead, your "
    "settled conclusion a finding, a named person or system an "
    "entity, a bare indicator an observable, a thing that "
    "happened an event — and let the suggestion stand exactly as "
    "the filing suggestion stands. The operator's words naming "
    "a part govern over any suggestion, yours included. A case can also be BORN from a Wire story (v0.6.6e, #330): the story's \"Start a case\" button, or the operator's words (\"start an investigation on that story\"), pull it together — article shelved, a cited lead per CVE it names, related coverage shelved, the case set open — and the case chat opens on a briefing the bridge composes from the case's live state, with your recommended next tasks staged as proposals awaiting the operator's approval. Facts drawn from "
    "a Wire record keep the record's values — its CVE id, CVSS, "
    "EPSS and dates; a filing that contradicts its cited record is "
    "flagged on its face by the bridge before it lands.")

# ---------------- #240: the intent gate + executor hygiene ----------
# v0.4.6e (#240): the #232 shape recurred in the field on
# v0.4.3e — a plain knowledge question answered by driving the
# shell (prose typed into bash, then invented evidence narrated
# back), once locally and once in a BORROWED reply that drove
# the borrower's session. The #232 prose guard treated the
# symptom (a hollow reply) but nothing decided whether a drive
# should happen at all. Three layers now stand in front of the
# terminal:
#
# 1. THE GATE (below): a block executes only on a
#    machine-directed turn — the composer latched to CLI mode,
#    a live session already open in this chat (the operator is
#    mid-flow in it), or the operator's own message carrying
#    machine intent. Otherwise the block is inert: stripped
#    like all markup, never typed, and the turn is answered in
#    conversation. The gate is deliberately PERMISSIVE toward
#    naturally-phrased machine asks (the operator's ruling: a
#    false refusal is friction, and friction is the enemy) —
#    it exists to stop conversation from reaching bash, not to
#    make the operator phrase his work a special way.
# 2. EXECUTOR HYGIENE (_shell_line_prose, in the drive loop):
#    lines are no longer typed verbatim — a leading "$ " is
#    stripped, and a line that reads as a sentence rather than
#    a command is refused and NAMED in the session record
#    instead of reaching the terminal.
# 3. THE GROUNDED GUARD (prose_guard, below): the re-ask
#    carries the session's REAL captured output, so the
#    narration can only be about what the terminal showed.
# The gate is computed at the one extraction site every reply
# passes through — local or mesh-served, the rules are the
# same; routing does not change them.

_SHELL_INTENT_NOUNS = re.compile(
    r"\b(disk|drive|ssd|hdd|ram|cpu|gpu|swap|process|processes|"
    r"service|services|systemd|journal|file|files|folder|folders|"
    r"directory|directories|path|paths|port|ports|network|wifi|"
    r"ethernet|terminal|shell|bash|command|commands|script|"
    r"scripts|docker|container|containers|kernel|mount|mounted|"
    r"partition|partitions|volume|uptime|package|packages|repo|"
    r"repository|git|server|machine|box|computer|laptop|desktop|"
    r"linux|distro|permission|permissions|ownership|cron|backup|"
    r"backups|firewall|bluetooth|printer|usb|device|devices|"
    r"driver|drivers|boot|startup|login|session|log|logs|"
    r"hostname|battery|fan|temperature|home folder|ip address)\b",
    re.I)
_SHELL_INTENT_VERBS = re.compile(
    r"\b(run|execute|install|uninstall|reinstall|kill|restart|"
    r"restarts|reboot|list|fix|update|upgrade|download|ping|"
    r"ssh|mount|umount|chmod|chown|grep|tail|cat|curl|wget|"
    r"watch|monitor|show me|look at|look in|look under)\b", re.I)
_SHELL_INTENT_SHAPES = re.compile(
    r"(/[\w.\-]{2,}|~/|\./|\b\w+\.(log|txt|py|sh|json|conf|cfg|"
    r"md|service|timer|sock)\b|--[\w\-]+|\s-[a-zA-Z]{1,2}\b|"
    r"`[^`]+`)", re.I)
_SHELL_INTENT_QUESTIONS = re.compile(
    r"(what'?s (eating|using|taking|hogging|running)|"
    r"how much (space|room|ram|memory|disk)|how (full|big) is|"
    r"why (is|does|did) .{0,40}(slow|full|failing|failed|down|"
    r"stuck|crashing)|is .{0,30}(running|installed|enabled|up|"
    r"down)\b|on (this|the|my) (box|machine|computer|server)|"
    r"my (disk|drive|machine|computer|server|files|system|box|"
    r"home folder))", re.I)


def shell_machine_intent(text):
    """Does the operator's message carry machine intent — is
    this turn about the machine, its files, processes, or
    services, or an imperative to run/show/check/fix something
    on it? Deliberately permissive (see the gate note above):
    any one signal qualifies. Plain knowledge questions —
    Spartan0311's field cases, the capital of France, the
    tallest mountain in Arizona — carry none of them."""
    t = (text or "").strip()
    if not t:
        return False
    return bool(
        _SHELL_INTENT_NOUNS.search(t)
        or _SHELL_INTENT_VERBS.search(t)
        or _SHELL_INTENT_SHAPES.search(t)
        or _SHELL_INTENT_QUESTIONS.search(t))


def shell_turn_machine_directed(user_text, chat_id, chat_mode=None):
    """The #240 gate, whole: a [[shell]] block in the reply to
    THIS turn may execute only when the turn is
    machine-directed — the composer latched to CLI mode (the
    operator's standing instruction), a live session already
    open in this chat (a follow-up inside its working flow),
    or the operator's message itself carrying machine intent.
    Computed once, at the extraction site, for local and
    mesh-served replies alike."""
    if chat_mode == "cli":
        return True
    if chat_id is not None and shell_session_get(chat_id) is not None:
        return True
    return shell_machine_intent(user_text)


_SHELL_PROSE_SENTENCE = re.compile(
    r"^[A-Z][a-z]+( [A-Za-z,;:’']+){3,}[.!?]$")
_SHELL_PROSE_COPULA = re.compile(
    r"\b(is|are|was|were|has|have|had)\b", re.I)
_SHELL_COMMAND_CHARS = re.compile(r"[/=|><$`'\";&()\\{}\[\]*?~#]")
_SHELL_KNOWN_COMMANDS = frozenset(
    "echo printf ls cat cd pwd whoami hostname date uptime df du "
    "free ps top htop systemctl journalctl grep tail head find "
    "ssh ping curl wget id uname env export source nano vi vim "
    "less more wc sort uniq awk sed xargs kill pkill mkdir rmdir "
    "touch cp mv rm ln chmod chown tar zip unzip git python "
    "python3 node npm pip pip3 docker ip ss netstat mount umount "
    "sudo su clear exit history which type man apt apt-get dnf "
    "pacman lsblk dmesg lscpu lsusb nproc who w last crontab ufw "
    "nmap dig nslookup traceroute file stat readlink basename "
    "dirname tee jq yq systemctl service".split())


def _shell_line_prose(line):
    """Executor hygiene (#240, layer 2): does this line read as
    a SENTENCE rather than a command? Two shapes convict: a
    capitalized sentence of four-plus words ending in sentence
    punctuation, and a five-plus-word line built around a
    copula (is/are/was/has…) with no command characters in it
    at all. Command lines — flags, paths, pipes, quotes,
    assignments — carry command characters or lack the
    sentence shape, and pass; a line that OPENS with a known
    command word (echo Paris is the capital…) is a command
    however sentence-like its arguments read. Judged AFTER
    the "$ " strip."""
    t = (line or "").strip()
    if not t:
        return False
    words = t.split()
    first = words[0].lower() if words else ""
    if first in _SHELL_KNOWN_COMMANDS:
        return False
    if first == "sudo" and len(words) > 1 \
            and words[1].lower() in _SHELL_KNOWN_COMMANDS:
        return False
    if _SHELL_PROSE_SENTENCE.match(t):
        return True
    if len(words) >= 5 and _SHELL_PROSE_COPULA.search(t) \
            and not _SHELL_COMMAND_CHARS.search(t):
        return True
    return False


def extract_shell_block(reply, chat_id, gate_open=True):
    """Navi drives the chat's interactive session: a [[shell]]
    block in her reply is stripped from the text, each line typed
    into the session (born here if this is her first interactive
    need), badged "navi". Returns
    (clean_reply, drove_or_None).

    v0.4.6e (#240): the drive is GATED — with gate_open False
    the markup is consumed exactly as always (it is
    presentation, never content) but nothing is typed and no
    session is born; the returned record says so
    ({"lines": [], "gated": True}) so the prose guard can
    re-ask in conversation. And the lines that DO drive pass
    executor hygiene first: a leading "$ " is stripped, and a
    line that reads as prose is refused — named in the session
    record when a session stands, and carried on the returned
    record ("refused") either way — never typed.

    Build 6 (#217): the markup is consumed in every shape a
    generation actually produces, because a borrowed reply showed
    the raw token in the field. The block match is
    case-insensitive (a model that closes with [[/Shell]] still
    drove, and still showed the token); more than one block in a
    reply is consumed in order; and an UNTERMINATED block — the
    generation was cut before the closing token, the borrowed
    shape — is cut from the text at the opening token and drives
    NOTHING: a truncated last line is half a command, and half a
    command is never typed. The drop is named in the returned
    record ("unterminated") instead of rendering as raw markup.
    A reply with no chat to drive (chat_id None) still has its
    markup consumed — it simply drives nothing: the token is
    presentation, never content."""
    if not reply:
        return reply, None
    if not gate_open:
        # #240, layer 1: the turn is not machine-directed. The
        # markup is consumed (presentation, never content) and
        # NOTHING is typed — no session is born on a
        # conversation turn. The record says the block was
        # gated, so the prose guard knows the shape it is
        # re-asking about: nothing ran.
        if not _SHELL_BLOCK_OPEN.search(reply):
            return reply, None
        return _strip_shell_blocks(reply), {"lines": [], "gated": True}
    drove = []
    refused = []
    drove_any = False
    out = reply
    settled = False
    for _ in range(3):
        m = _SHELL_BLOCK.search(out)
        if not m:
            break
        lines = m.group(1).split("\n")
        while lines and lines[0] == "":
            lines.pop(0)
        while lines and lines[-1] == "":
            lines.pop()
        for ln in lines[:20]:
            if chat_id is None:
                continue
            # #240, layer 2 (executor hygiene): a leading "$ "
            # is the prompt she copied, not part of the command
            # — strip it. A line that reads as prose is refused
            # here, never typed (named after the loop).
            clean_ln = ln[2:] if ln.startswith("$ ") else ln
            if _shell_line_prose(clean_ln):
                refused.append(clean_ln)
                continue
            if not settled:
                # v0.4.2e (#232): a drive that BIRTHS the
                # session waits for the startup stream to
                # settle before its first line is typed, so
                # the banner precedes the drive in the record
                # instead of flooding it.
                settled = True
                if shell_session_get(chat_id) is None:
                    try:
                        shell_session_open(chat_id).wait_settled()
                    except Exception:
                        pass
            if shell_session_input(
                    chat_id, clean_ln + "\n", "navi", create=True):
                drove.append(clean_ln)
        drove_any = True
        out = (out[:m.start()] + out[m.end():]).strip()
    if refused:
        # The refusals are named in the session record when a
        # session stands (pre-existing, or born by a line that
        # did type). When nothing typed and no session stands,
        # no session is born just to hold the note — the record
        # below carries the refused lines and the guard names
        # them to her.
        sess = shell_session_get(chat_id) if chat_id is not None \
            else None
        if sess is not None:
            for ln in refused:
                try:
                    sess._emit(
                        "note", None,
                        "refused to type a line that reads as "
                        "prose, not a command: " + ln)
                except Exception:
                    pass
    mo = _SHELL_BLOCK_OPEN.search(out)
    if mo:
        out = out[:mo.start()].strip()
        rec = {"lines": drove, "unterminated": True}
        if refused:
            rec["refused"] = refused
        return out, rec
    if not drove_any:
        return reply, None
    rec = {"lines": drove}
    if refused:
        rec["refused"] = refused
    return out, rec


def _strip_shell_blocks(text):
    # The [[shell]] markup removed WITHOUT driving anything —
    # for a text that must never reach a session (the prose
    # guard's re-ask, below). The same consumption shapes as
    # extract_shell_block: paired blocks out, an unterminated
    # block cut at its opening token.
    if not text:
        return text
    out = _SHELL_BLOCK.sub("", text)
    mo = _SHELL_BLOCK_OPEN.search(out)
    if mo:
        out = out[:mo.start()]
    return out.strip()


def prose_guard(reply, drove, messages, endpoint, model,
                chat_id=None):
    """v0.4.2e (#232): a model turn must still ANSWER. The
    field shape, twice in one afternoon: a conversational
    question answered with a bare [[shell]] block (extraction
    left the prose empty and the terminal block stood as the
    whole reply), and a machine question answered by reading
    an injected brief back. When the cleaned reply is empty,
    she is re-asked ONCE — the failure named, every [[...]]
    block forbidden in the re-ask (a shell block in it is
    stripped, never driven) — and if the re-ask still says
    nothing, a plain bridge sentence names what was run, so
    the operator is never handed a hollow reply and left to
    guess what happened. A reply with any prose at all passes
    through untouched. Returns the reply text to ship.

    v0.4.6e (#240, layer 3): the re-ask is GROUNDED. The
    #232 re-ask named what she ran but never showed her what
    the session actually printed, so the narration was
    invented to fill the gap (a README that never existed,
    offered as evidence). Now the re-ask carries the session's
    REAL captured output — errors included — and binds her to
    it: narrate only what the output shows; if nothing ran
    (a gated block, refused lines), the re-ask says nothing
    ran, and so must she."""
    if (reply or "").strip():
        return reply
    lines = list((drove or {}).get("lines") or [])
    refused = list((drove or {}).get("refused") or [])
    gated = bool((drove or {}).get("gated"))
    shown = ""
    looked = False
    if lines and chat_id is not None:
        # Let the drive's output land, then read what the
        # session ACTUALLY showed — the only ground the
        # narration may stand on.
        sess = shell_session_get(chat_id)
        if sess is not None:
            looked = True
            try:
                sess.wait_settled(quiet=0.4, cap=2.0, floor=0.0)
            except Exception:
                pass
            try:
                shown = (sess.output_tail(1500) or "").strip()
            except Exception:
                shown = ""
    note = "Your previous reply to the operator "
    if gated:
        note += ("tried to drive this chat's shell on a "
                 "conversational turn, so the bridge did NOT "
                 "run anything — nothing was typed, nothing "
                 "executed. ")
    elif lines:
        note += ("drove this chat's shell ("
                 + ", ".join("`" + ln + "`" for ln in lines)
                 + ") but ")
    if refused:
        note += ("(Lines refused as prose and never typed: "
                 + ", ".join("`" + ln + "`" for ln in refused)
                 + ".) ")
    note += ("said nothing in words. Answer him now, in prose, "
             "in character: say what you ran and what it is "
             "for, or answer his question directly. Do NOT emit "
             "a [[shell]] block or any other [[...]] block in "
             "this reply.")
    if lines and looked:
        if shown:
            note += ("\n\nThe shell session actually showed "
                     "this — narrate ONLY what it shows, never "
                     "a file, path, or result that is not in "
                     "it:\n" + shown)
        else:
            note += ("\n\nThe shell session has shown nothing "
                     "back — say plainly that nothing came "
                     "back, and never invent a result.")
    elif gated or refused:
        note += ("\n\nNothing ran. Do not describe any command, "
                 "output, or result — answer his question from "
                 "what you know, in words.")
    again = None
    try:
        again = llamafile_chat(
            list(messages) + [{"role": "system", "content": note}],
            endpoint, model)
    except Exception:
        again = None
    if again:
        again = _strip_shell_blocks(strip_citation_footer(again))
        if (again or "").strip():
            return again
    if lines:
        return ("I ran " + ", ".join("`" + ln + "`" for ln in lines)
                + " in this chat's shell — its output is in the "
                  "thread, and I'll speak to what it shows.")
    if refused:
        return ("I caught myself about to type a sentence "
                "into the shell instead of answering you — "
                "nothing was typed and nothing ran. Ask me "
                "again and I'll answer in words.")
    return ("That one came back empty on my side — ask me again "
            "and I'll answer properly.")


def shell_session_brief(chat_id):
    """What Navi is told about this chat's live session before she
    answers: that it lives, and the tail of what it has shown."""
    if chat_id is None:
        return ""
    sess = shell_session_get(chat_id)
    if sess is None:
        return ""
    tail = sess.output_tail(1200)
    out = ("A live interactive shell session is open in this chat. "
           "Its recent output:\n" + (tail or "(nothing yet)"))
    if sess.grid.in_alt:
        out += ("\n(A full-screen program currently holds the "
                "terminal.)")
    return out


def installed_commands():
    # What this machine actually has installed: every executable on
    # PATH. This is the list command autocomplete draws from in live
    # mode (plus the built-in default set the dashboard always offers).
    names = set()
    for directory in os.environ.get("PATH", "").split(os.pathsep):
        if not directory:
            continue
        try:
            entries = os.listdir(directory)
        except Exception:
            continue
        for name in entries:
            if not re.match(r"^[A-Za-z0-9][A-Za-z0-9+._-]*$", name):
                continue
            path = os.path.join(directory, name)
            if os.path.isfile(path) and os.access(path, os.X_OK):
                names.add(name)
    return sorted(names)[:800]


# ------------------------- CLI-first routing -------------------------
# Chat is CLI-first: a submitted line whose first word is a system
# command runs immediately — no model call — and the raw output
# lands in the thread as a terminal block, where the dashboard folds
# it into the history the model reasons over on later turns. The
# command universe is the same one autocomplete offers: the built-in
# default set plus every executable on PATH (installed_commands,
# cached for a minute so a chat send never pays a full PATH scan
# more than that). A leading "$" forces the shell; a leading "> "
# forces the line to Navi. Under them sits the composer's latched
# mode (the dashboard's CLI / Navi press-once toggles, sent as
# "mode"): an unmarked line routes the latched way, and only with
# no latch does the first-word check decide. Chips are routed
# before this is consulted and are never eaten by it.

DEFAULT_COMMAND_NAMES = {
    "ls", "cat", "cd", "pwd", "echo", "whoami", "hostname", "date",
    "uname", "ps", "navi", "help",
}
_commands_cache = {"at": 0.0, "names": None}


def known_commands():
    now = time.time()
    if _commands_cache["names"] is None or now - _commands_cache["at"] > 60:
        _commands_cache["names"] = set(installed_commands()) | DEFAULT_COMMAND_NAMES
        _commands_cache["at"] = now
    return _commands_cache["names"]


def cli_route(text, mode=None):
    # Returns (command, navi_text) — exactly one is not None: the
    # shell command to run, or the text the model should see (with a
    # forced-Navi "> " marker stripped). The mode argument is the
    # composer's latched mode: it decides unmarked lines, markers win.
    t = (text or "").strip()
    if t.startswith("$"):
        return t[1:].strip(), None
    if t == ">" or t.startswith("> "):
        return None, t[1:].strip()
    if mode == "cli":
        return t, None
    if mode == "navi":
        return None, t
    body = t
    if body == "sudo":
        body = ""
    elif body.startswith("sudo "):
        body = body[5:].lstrip()
    parts = body.split()
    first = parts[0].lower() if parts else ""
    if "/" in first:
        first = first.rsplit("/", 1)[-1]
    if first and first in known_commands():
        return t, None
    return None, t


# ------------------------- Chip management -------------------------
# The Chips screen has two tabs. Navi User Repo (NUR) searches GitHub for
# every public repo carrying the "navi-chips" topic — each one is a
# Navi chip — and tracks installs in chips/installed.json: install
# downloads the chip's own source from the repo (at its latest
# release, or its default branch when it has no releases yet) and
# forges it into a chip in chips/, so an installed chip is a
# chip like any other — on the pill row, runnable from chat,
# executable by the bridge; delete removes the record and the
# chip; and an update is
# available when the repo's latest release tag differs from the
# installed one (or, for release-less repos, when the repo has been
# pushed to since the install) — updating re-downloads the source
# and re-forges the chip. Chip Studio forges new chips into
# .png chips — the chip's whole codebase embedded in a PNG tEXt
# chunk, PICO-8 style — saved into chips/ and registered as local
# installs the Navi User Repo tab lists alongside the GitHub ones.

CHIPS_DIR = os.path.join(ROOT, "chips")
CHIP_INSTALLS_PATH = os.path.join(CHIPS_DIR, "installed.json")
GITHUB_HEADERS = {
    "Accept": "application/vnd.github+json",
    "User-Agent": "navi-apex-bridge",
}

# ---------------------------------------------------------------------
# Sign-in Build 2 (0.2.0e, #173): failures with names, and the
# reads' own credential.
#
# Every GitHub failure Apex surfaces is classified into one of
# three named kinds — auth (the credential itself is refused),
# rate (a bucket is spent; the reset time rides along), abuse
# (GitHub's secondary/abuse safeguard; Retry-After rides along and
# Apex stops — nothing anywhere retries on its own) — and worded
# as a sentence. No surface ever shows a bare "HTTP 403" again:
# that silence is what made the field 403s (#172) undiagnosable.
#
# And the app reads — the Updater's checks, the NUR's topic
# searches and repo reads, downloads — ride the operator's own
# GitHub sign-in (Build 6, #173 phase 3: one credential for the
# whole product; Build 2's separate Runtime PAT is retired).
# Signed out, they go unauthenticated and share that bucket —
# 60 requests/hour per network address, 10/minute for search —
# with everything else on the operator's network, and their
# failures name those limits out loud.


class GithubHttpError(Exception):
    """A GitHub HTTP failure with its evidence attached: status,
    GitHub's own message, and the response headers (rate-limit and
    Retry-After facts live there). .code mirrors HTTPError's, so
    older callers that read it keep working."""

    def __init__(self, status, message="", headers=None, url=""):
        super().__init__("HTTP " + str(status) + ((": " + message) if message else ""))
        self.status = status
        self.code = status
        self.message = message or ""
        self.headers = headers
        self.url = url


def _gh_header(headers, name):
    if headers is None:
        return None
    try:
        v = headers.get(name)
    except Exception:
        return None
    return str(v) if v is not None else None


# The last rate-limit buckets GitHub actually reported, keyed by
# resource ("core" / "search"). Filled in from response headers
# wherever a GitHub response provides them; the Settings → GitHub
# identity panel reads them back. None until a response lands.
_GITHUB_RATE = {"core": None, "search": None}


def _github_note_rate(headers, url=""):
    try:
        limit = _gh_header(headers, "x-ratelimit-limit")
        if limit is None:
            return
        resource = (_gh_header(headers, "x-ratelimit-resource") or "").strip().lower()
        if resource not in ("core", "search"):
            resource = "search" if "/search/" in (url or "") else "core"
        remaining = _gh_header(headers, "x-ratelimit-remaining")
        reset = _gh_header(headers, "x-ratelimit-reset")
        _GITHUB_RATE[resource] = {
            "limit": int(limit),
            "remaining": int(remaining) if remaining is not None else None,
            "reset": int(reset) if reset is not None else None,
            "at": int(time.time()),
        }
    except Exception:
        pass


def github_rate_state():
    return {"core": _GITHUB_RATE.get("core"), "search": _GITHUB_RATE.get("search")}


def _gh_reset_phrase(reset_epoch):
    if not reset_epoch:
        return ""
    try:
        when = time.strftime("%H:%M", time.localtime(int(reset_epoch)))
    except Exception:
        return ""
    return " It resets at " + when + "."


def github_classify_error(status, headers=None, message="", authed=True):
    """Classify one GitHub failure and word it. Returns a dict:
    kind ("auth" | "rate" | "abuse" | "error"), the HTTP status,
    GitHub's own message where it sent one, the reset time and
    Retry-After where the headers carry them, and `text` — the
    full sentence a surface shows. `authed` says whether the
    request carried a credential; an unauthenticated rate failure
    names the unauthenticated limits explicitly, because that is
    the fact that fixes it."""
    msg = (message or "").strip()
    ml = msg.lower()
    retry_after = _gh_header(headers, "retry-after")
    try:
        retry_after_s = int(retry_after) if retry_after is not None else None
    except Exception:
        retry_after_s = None
    reset_raw = _gh_header(headers, "x-ratelimit-reset")
    try:
        reset_at = int(reset_raw) if reset_raw is not None else None
    except Exception:
        reset_at = None
    remaining = _gh_header(headers, "x-ratelimit-remaining")
    kind = "error"
    if status in (401,):
        kind = "auth"
    elif status == 429 or "secondary rate" in ml or "abuse detection" in ml or (status == 403 and retry_after_s is not None):
        kind = "abuse"
    elif status == 403 and (remaining == "0" or "rate limit" in ml):
        kind = "rate"
    elif status == 403:
        kind = "auth"
    if kind == "auth":
        if status == 401:
            text = ("GitHub refused the sign-in itself (HTTP 401) — the credential is missing, "
                    "expired, or revoked. This is not a rate limit; waiting will not cure it.")
        else:
            text = ("GitHub answered 403 Forbidden, and it is not a rate limit — the credential "
                    "lacks access to this, or GitHub is refusing it on sight. Waiting will not cure it.")
    elif kind == "rate":
        text = "GitHub's rate limit is used up (HTTP " + str(status) + ")." + _gh_reset_phrase(reset_at)
        if not authed:
            text += (" This request went out unauthenticated, so it shared the unauthenticated "
                     "allowance — 60 requests per hour per network address (10 per minute for "
                     "search). Sign in with your GitHub account in Settings → GitHub and the app's "
                     "own reads ride your own allowance instead.")
    elif kind == "abuse":
        text = ("GitHub's abuse safeguard tripped (HTTP " + str(status) + ") — automated-looking "
                "traffic, in GitHub's judgment.")
        if retry_after_s is not None:
            text += " It asked Apex to wait " + str(retry_after_s) + " seconds."
        text += " Apex stops here; nothing retries on its own."
    elif status and status >= 500:
        text = "GitHub itself is having trouble (HTTP " + str(status) + ") — the fault is on their side; try again later."
    elif status == 404:
        text = "GitHub answered 404 — what was asked for is not there (or is private to another account)."
    else:
        text = "GitHub answered HTTP " + str(status) + ((": " + msg) if msg else ".")
    return {
        "kind": kind, "status": status, "message": msg,
        "resetAt": reset_at, "retryAfterSeconds": retry_after_s,
        "unauthenticated": not authed, "text": text,
    }


def github_error_info(e, authed=True):
    """The classification for a caught failure: GithubHttpError
    and urllib's HTTPError both carry their evidence; anything
    else (a socket that never connected) is worded as reach."""
    if isinstance(e, GithubHttpError):
        return github_classify_error(e.status, e.headers, e.message, authed=authed)
    if isinstance(e, urllib.error.HTTPError):
        msg = ""
        try:
            body = e.read()
            if body:
                parsed = json.loads(body.decode("utf-8", "replace"))
                if isinstance(parsed, dict):
                    msg = str(parsed.get("message") or "")
        except Exception:
            pass
        return github_classify_error(e.code, getattr(e, "headers", None) or getattr(e, "hdrs", None), msg, authed=authed)
    if isinstance(e, urllib.error.URLError):
        return {"kind": "error", "status": None, "message": str(e),
                "resetAt": None, "retryAfterSeconds": None, "unauthenticated": not authed,
                "text": "GitHub could not be reached at all — no connection, or github.com is down."}
    return {"kind": "error", "status": None, "message": str(e),
            "resetAt": None, "retryAfterSeconds": None, "unauthenticated": not authed,
            "text": "The GitHub request failed: " + str(e)}


def github_error_text(e, authed=True):
    return github_error_info(e, authed=authed)["text"]


def _now_iso():
    return time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())


def _github_read_headers(accept=None):
    headers = dict(GITHUB_HEADERS)
    if accept:
        headers["Accept"] = accept
    token = github_read_token()
    if token:
        headers["Authorization"] = "Bearer " + token
    return headers, bool(token)


def _github_read(url, accept=None):
    # One app-runtime read (Updater / NUR), behind Build 11's
    # renewal wrapper (#197): the stored token is renewed
    # BEFORE the call when its recorded expiry says it is dead
    # or nearly so, and a 401 that still comes back earns
    # exactly one renewal + one retry. A 401 surviving that
    # raises the same GithubHttpError it always did — the
    # named auth sentence, once.
    _auth, _attempted = github_ensure_fresh_token()
    try:
        return _github_read_once(url, accept)
    except GithubHttpError as e:
        if e.code == 401 and not _attempted and github_refresh_auth():
            return _github_read_once(url, accept)
        raise


def _github_read_once(url, accept=None):
    # The single attempt. Rides the operator's sign-in token;
    # failures raise GithubHttpError with the response's
    # headers attached, so the caller can classify instead of
    # guessing from a status code.
    headers, _authed = _github_read_headers(accept)
    req = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=20) as resp:
            _github_note_rate(resp.headers, url)
            return resp.read()
    except urllib.error.HTTPError as e:
        _github_note_rate(e.headers, url)
        msg = ""
        try:
            body = e.read()
            if body:
                parsed = json.loads(body.decode("utf-8", "replace"))
                if isinstance(parsed, dict):
                    msg = str(parsed.get("message") or "")
        except Exception:
            pass
        raise GithubHttpError(e.code, msg, e.headers, url)


def github_json(url):
    return json.loads(_github_read(url).decode())


def github_text(url, accept):
    return _github_read(url, accept).decode("utf-8", "replace")


# Topic-search discipline (Build 2, #173): the NUR's two catalog
# searches share one cache and one road — results are cached for
# five minutes, and a second search waits for the one in flight
# instead of doubling the Search bucket's spend (10/minute
# unauthenticated is the tightest allowance Apex touches).
_GITHUB_SEARCH_CACHE = {}
_GITHUB_SEARCH_LOCK = threading.Lock()
_GITHUB_SEARCH_TTL = 300


def github_search_repositories(query):
    now = time.time()
    hit = _GITHUB_SEARCH_CACHE.get(query)
    if hit and hit[0] > now:
        return hit[1]
    with _GITHUB_SEARCH_LOCK:
        hit = _GITHUB_SEARCH_CACHE.get(query)
        if hit and hit[0] > time.time():
            return hit[1]
        data = github_json("https://api.github.com/search/repositories?q=" + query + "&per_page=30&sort=updated")
        _GITHUB_SEARCH_CACHE[query] = (time.time() + _GITHUB_SEARCH_TTL, data)
        return data


# ---------------------------------------------------------------------
# App updates — the Updater, for real (build .47; end-to-end apply
# in build .56).
#
# A check asks GitHub for the project's latest published release
# and compares its tag against the VERSION file (the very number
# app_version reports), so the footer, the status answer, and the
# Updater all read the same truth. A check that cannot complete —
# offline, or GitHub's rate limit — answers ok: False with the
# reason named. It never dresses a failure up as "up to date".
#
# Apply carries the update through from the one click (build .56):
# download the release's package zip -> verify it -> stage it
# under updates/<tag>/package -> hand the install to a detached
# helper the bridge writes into updates/<tag>/, which outlives the
# bridge, runs the staged installer itself (never --fresh: the
# installer is the one path that replaces the app, because it owns
# the state-preservation stash — chips, packs and their indexes,
# the llamafile, Soul/Memory, voice, GitHub settings — and the
# bridge cannot overwrite itself while it runs) -> relaunch the
# app through start.sh -> confirm. The helper records every step
# in config/update-result.json; the panel polls that record while
# the apply runs (downloading / verifying / staging / installing /
# restarting) and reads the outcome from it afterwards. "Updated"
# is claimed only when the install after the relaunch reads as
# the target — the VERSION file's number AND, for a same-version
# Edge refresh, the build sequence the relaunched bridge reports
# and the one baked into the bridge.py on disk (Build 4: the
# number alone is the same before and after a refresh, so a
# confirmation that read only the number once certified an
# install of the previous build as the new one). A step that did
# not observe its own success never reports one.
#
# The failure ladder is part of the design: every step can fail
# and names itself and its reason in the record (download failed,
# not a zip, not an Apex package, staging failed, the installer
# failed with its exit state, the relaunch did not answer, the
# version after relaunch is not the expected one). A failed
# validation leaves nothing staged; a failure at or after staging
# leaves the staged package in place and the record carries the
# manual fallback — the staged folder and the one install
# command — labeled as the fallback it is.
#
# Build 6 — the helper can no longer die traceless. The Build 4
# round-trip gate ran a genuine Build 3 install applying a
# package through its own Updater: the staged installer ran
# through to "Done.", and then the detached helper died without
# a trace — apply.log 0 bytes, no apply-relaunch.log ever
# created, the record frozen at working/installing, no bridge
# relaunched. Diagnosis from the helper's source: its step
# sequence after the installer exits is read the install log
# back -> record the DHD-prerequisites step -> record
# "restarting" -> spawn start.sh -> poll /api/status for up to
# five minutes -> verify (Build 4's lock: the answering bridge
# reports the target build AND bridge.py on disk carries it)
# -> finish the record. Every death in that window was
# invisible by construction: the helper printed nothing (so
# apply.log is empty BY DESIGN on any run that does not crash
# Python itself), it had no top-level guard and no signal
# handlers, write_result merged into the record with no
# freshness stamp, and the failure-path ordering called fail()
# BEFORE relaunch() — an exception inside the failure write
# itself skipped the relaunch entirely. Ranked mechanisms for
# the gate's evidence: (1) an external signal kill at the
# installer's end — the only mechanism consistent with all
# three facts at once (a Python exception would have left a
# traceback in apply.log; the log was empty; a mere hang would
# have left a live process). The helper runs in its own
# session but in the same process tree and user-service world
# as everything the installer stops and restarts at exactly
# that moment — pattern kills, the pid-file kill, the shell
# stop, service manipulation — and SIGTERM/SIGKILL leave no
# trace in the victim. (2) An uncaught exception in the
# unguarded post-installer segment (record writes against a
# config tree the installer had just replaced/re-owned) —
# real, and closed by the guard, but it predicts a traceback
# the gate did not see, so it ranks second for THIS incident.
# (3) A block inside subprocess.run past the installer's
# "Done." — indistinguishable from death in the old design,
# which is itself the indictment. The repair defends against
# all three and makes the next incident self-convicting: the
# helper keeps a step log (updates/<tag>/apply-trace.log) from
# its first instruction — step boundaries, the installer's
# exit, the relaunch spawn, the confirmation's found-vs-
# expected values, a 30-second heartbeat (a log still beating
# at one step is a block; a log whose beats stop is a kill) —
# a top-level guard records any exception into the record
# (failed + step + reason) and mirrors the traceback to
# apply.log, signal handlers record SIGTERM/SIGINT/SIGHUP the
# same way before exiting, a relaunch that cannot even spawn
# fails loudly at "restart" with found-vs-expected detail, and
# the no-answer failure names what was expected and what was
# found. For the death no in-process guard survives (SIGKILL,
# OOM, power), the bridge's apply watchdog closes a record
# whose helper pid is dead and whose last write is stale as
# failed at its last step, quoting the step log's final line —
# the record can never again freeze at working/installing.
#
# Gzip tolerance (build .56): a package asset can arrive
# gzip-wrapped (its first bytes are the gzip signature, 1f 8b,
# instead of a zip's PK) when whatever staged the release asset
# compressed the zip itself. The validator unwraps that layer
# and runs the same Apex-package verification on the zip inside;
# only when neither the raw payload nor its unwrapped form is a
# valid Apex package does it refuse, and the refusal names what
# was wrong. The plain-PK path behaves exactly as it always did.
#
# NAVI_UPDATE_FEED_URL overrides the release-feed address. It is
# the seam the project's own end-to-end updater tests use to run
# this real apply path against a local stand-in instead of the
# live release; it is unset in normal installs.

UPDATES_DIR = os.path.join(ROOT, "updates")
UPDATE_RESULT_PATH = os.path.join(ROOT, "config", "update-result.json")


def _version_tuple(raw):
    # "v0.2.0" / "0.2.0-beta" -> (0, 2, 0): the numeric core is what
    # orders releases. A leading v and any pre-release word are
    # ignored for ordering; a tag that will not parse orders as
    # (0,), below everything real.
    text = str(raw or "").strip().lower()
    if text.startswith("v"):
        text = text[1:]
    text = text.split("-")[0].split("+")[0]
    parts = []
    for piece in text.split("."):
        digits = ""
        for ch in piece:
            if ch.isdigit():
                digits += ch
            else:
                break
        parts.append(int(digits) if digits else 0)
    return tuple(parts) or (0,)


def _version_newer(candidate, current):
    a, b = _version_tuple(candidate), _version_tuple(current)
    width = max(len(a), len(b))
    a += (0,) * (width - len(a))
    b += (0,) * (width - len(b))
    return a > b


def _status_rank(status):
    # Fuller beats preview at the same numbers: a full release
    # outranks beta, which outranks an Edge build.
    return {"stable": 2, "beta": 1, "edge": 0}.get(status, 0)


def _release_is_newer(tag, current_number, current_status, build_seq=None):
    # The Updater's real comparison (build .83): higher numbers
    # win; at the same numbers, only a graduation counts — an
    # Edge build of the numbers you already run as a full release
    # is a preview you passed, not an update, and the full
    # release of numbers you run as an Edge build IS one.
    # Build .86 adds the third tier: same numbers AND same status
    # AND the running build is an Edge publication — a candidate
    # carrying a higher Build number (build_seq) than the running
    # SHIPPED_BUILD_SEQ is a same-version refresh and IS newer; a
    # candidate with no build number, or one no higher, is NOT.
    num, status = _tag_parts(tag)
    if not num:
        return False
    a, b = _version_tuple(num), _version_tuple(current_number)
    width = max(len(a), len(b))
    a += (0,) * (width - len(a))
    b += (0,) * (width - len(b))
    if a != b:
        return a > b
    if _status_rank(status) != _status_rank(current_status):
        return _status_rank(status) > _status_rank(current_status)
    if current_status == "edge" and build_seq is not None:
        return build_seq > SHIPPED_BUILD_SEQ
    return False


def _release_build_seq(rel):
    # A publication's place in its own version's sequence
    # (build .86; the stamps token is #277): the release kit
    # stamps every build, and the stamps form "Stamps <date>
    # / seq <n>" — searched in the release's name first, then
    # its body's opening lines — is AUTHORITATIVE when
    # present: prose cannot outvote it. The legacy "Build
    # <n>" scan remains as a fallback for releases published
    # before the stamps line, but a "Build <n>" that is part
    # of a longer name never counts — the v0.5.7e notes'
    # opening named the feature "FediChat Build 2", the feed
    # read seq 2, and the identity check refused the genuine
    # seq-23 package on every Edge seat (#277). A match
    # immediately preceded by a letter + space is a name,
    # not a sequence. None when the release carries no
    # usable build number — the identity check then skips,
    # as it always has for unstamped releases.
    name = str(rel.get("name") or "")
    body = str(rel.get("body") or "")
    opening = [name] + body.splitlines()[:6]
    for text in opening:
        m = re.search(r"Stamps [0-9][0-9.\-]* / seq ([0-9]+)", text)
        if m:
            return int(m.group(1))
    for text in opening:
        for m in re.finditer(r"Build ([0-9]+)", text):
            if m.start() >= 2 and text[m.start() - 2].isalpha() \
                    and text[m.start() - 1] == " ":
                continue
            return int(m.group(1))
    return None


def _update_error_text(e):
    # A failed check, worded honestly (Build 2, #173): the failure
    # is classified and named — auth, rate (with its reset), abuse
    # (with its wait) — never a bare status code and never a guess.
    # The check rides the operator's sign-in when one exists;
    # the classifier hears which, so an unauthenticated rate
    # failure names the unauthenticated allowance out loud.
    if isinstance(e, GithubHttpError) and e.status == 404:
        return ("GitHub answered 404 — the project has no "
                "published release yet, so there is nothing to "
                "update to.")
    return github_error_text(e, authed=bool(github_read_token()))


def updater_channel_explicit(cfg=None):
    # True only when the OPERATOR picked a channel in the Updater
    # panel — the /api/settings/updater-channel save stamps
    # channelExplicit beside the choice (Build 2). A channel value
    # that arrived baked into the config by the package, or was
    # written by a build from before channels were a choice, is a
    # starting shape, not a choice, and never counts as one.
    if isinstance(cfg, dict):
        block = cfg.get("updater")
        if isinstance(block, dict):
            return block.get("channelExplicit") is True and block.get("channel") in ("stable", "edge")
    return False


def updater_channel(cfg=None):
    # The EFFECTIVE release channel this install follows (build
    # .83; the default rule repaired in Build 2). An explicit
    # operator choice wins, exactly as saved. With no explicit
    # choice, the running build's own line is the default: an
    # Edge build follows Edge — a config that predates channels
    # carries a baked "stable" it never chose, and honoring it
    # stranded Edge installs on a channel that never publishes
    # their builds (the Build 2 field report: an Edge install
    # told it was up to date against Stable's v0.1.7). A Stable
    # build's default stays "stable" — the safe channel is the
    # default for the line that has one, never silently Edge.
    if updater_channel_explicit(cfg):
        return cfg["updater"]["channel"]
    if app_version_status(cfg) == "edge":
        return "edge"
    return "stable"


def _shape_release(rel):
    # One GitHub release record in the Updater's shape. The
    # package asset is the release's .zip (the export package);
    # a release without one can be reported but never staged.
    asset = None
    for a in rel.get("assets") or []:
        name = str(a.get("name") or "")
        if name.lower().endswith(".zip"):
            asset = {
                "name": name,
                "size": a.get("size") or 0,
                "url": a.get("browser_download_url") or "",
            }
            break
    return {
        "tag": str(rel.get("tag_name") or ""),
        "name": str(rel.get("name") or ""),
        "url": str(rel.get("html_url") or ""),
        "publishedAt": str(rel.get("published_at") or ""),
        "notes": str(rel.get("body") or "").replace(chr(13) + chr(10), chr(10)).replace(chr(13), chr(10)),
        "prerelease": bool(rel.get("prerelease")),
        "buildSeq": _release_build_seq(rel),
        "asset": asset,
    }


def latest_release(channel="stable"):
    # The newest release on a channel, shaped for the Updater;
    # raises on any failure and lets the caller word it. Stable
    # asks GitHub for its "latest" — by GitHub's own rule the
    # newest non-prerelease, the main branch's line. Edge reads
    # the release list and takes the newest published entry,
    # pre-releases included — the dev branch's line, kept
    # separate from main, which is what an Edge build is.
    feed_url = os.environ.get("NAVI_UPDATE_FEED_URL") or ""
    if feed_url:
        rel = github_json(feed_url)
        if not isinstance(rel, dict):
            raise ValueError("GitHub's answer was not a release record.")
        return _shape_release(rel)
    if channel == "edge":
        items = github_json(GITHUB_API_BASE + "/releases?per_page=10")
        if isinstance(items, list):
            for item in items:
                if isinstance(item, dict) and not item.get("draft"):
                    return _shape_release(item)
        raise ValueError("GitHub's release list carried no published release.")
    rel = github_json(GITHUB_API_BASE + "/releases/latest")
    if not isinstance(rel, dict):
        raise ValueError("GitHub's answer was not a release record.")
    return _shape_release(rel)


def current_release(cfg=None):
    # The release page for the version this install is running —
    # the Updater's "About this release" dropdown reads it: name,
    # tag, date, link, and the full notes, from the release page
    # itself. Best-effort: any failure answers None and the panel
    # says the page could not be fetched; it never invents one.
    number = app_version(cfg)
    status = app_version_status(cfg)
    tags = [_version_label(number, status), "v" + str(number)]
    seen = []
    for t in tags:
        if t and t not in seen:
            seen.append(t)
    for tag in seen:
        try:
            rel = github_json(GITHUB_API_BASE + "/releases/tags/" + tag)
        except Exception:
            continue
        if isinstance(rel, dict) and rel.get("tag_name"):
            return _shape_release(rel)
    return None


def changelog_section_for(tag):
    # The CHANGELOG fallback for the Updater's release notes
    # (build .70, issue #47): when a release is published with an
    # empty body, the panel still deserves the notes, so the
    # bridge reads the project's CHANGELOG.md from the repo and
    # cuts out the section for the release's version — the
    # "## 0.1.6 (alpha)" heading down to the next "## " heading.
    # Any failure (offline, no such section) returns "" and the
    # panel simply shows no notes; it never invents any.
    number, _status = _tag_parts(tag)
    if not number:
        return ""
    try:
        text = github_text(
            "https://raw.githubusercontent.com/AlexKollar/Apex/main/CHANGELOG.md",
            "text/plain")
    except Exception:
        return ""
    lines = str(text).splitlines()
    start = None
    for i, line in enumerate(lines):
        if line.startswith("## "):
            head = line[3:].strip()
            head_number = head.split(" ")[0].split("(")[0].strip()
            if head_number == number:
                start = i
                break
    if start is None:
        return ""
    out = []
    for line in lines[start + 1:]:
        if line.startswith("## "):
            break
        out.append(line)
    return "\n".join(out).strip()


# -- Background update re-check (build .86) -------------------------
# While the bridge runs, a daemon thread re-checks for updates on
# its own: shortly after startup, then every 3 hours, on the
# install's configured channel. The verdict is cached here and
# /api/status carries it as its "update" object, so the dashboard
# learns about an update without the operator ever opening the
# Updater. Silent by design — no UI of its own, no nag; the
# status line's single "Up to date" / "Update available" voice is
# still the only voice.
UPDATE_BG_STATE = {"payload": None}
UPDATE_BG_LOCK = threading.Lock()


def update_bg_payload():
    with UPDATE_BG_LOCK:
        return UPDATE_BG_STATE["payload"]


def _update_bg_loop():
    time.sleep(20)
    while True:
        try:
            cfg = load_config()
            res = update_check_payload(cfg)
            payload = {
                "updateAvailable": bool(res.get("updateAvailable")),
                "latest": res.get("latest") or "",
                "channel": res.get("channel") or updater_channel(cfg),
                "checkedAt": res.get("checkedAt") or "",
            }
            with UPDATE_BG_LOCK:
                UPDATE_BG_STATE["payload"] = payload
        except Exception:
            pass
        time.sleep(3 * 3600)


def update_check_payload(cfg=None, channel=None):
    current = app_version(cfg)
    current_status = app_version_status(cfg)
    # Channel resolution (Build 2): a persisted explicit choice is
    # authoritative — the check follows the channel the operator
    # picked, and the payload says so (channelExplicit). With no
    # explicit choice, an Edge build checks Edge even when the
    # caller asks for "stable": the only caller that asks is a
    # dashboard whose baked starting channel predates the choice,
    # and that inherited word is not a choice (see
    # updater_channel). On a Stable build with no explicit choice,
    # the caller's channel is honored, else the effective default.
    if updater_channel_explicit(cfg):
        channel = updater_channel(cfg)
    elif current_status == "edge":
        channel = "edge"
    elif channel not in ("stable", "edge"):
        channel = updater_channel(cfg)
    base = {
        "current": current,
        "currentStatus": current_status,
        "channel": channel,
        "channelExplicit": updater_channel_explicit(cfg),
        "checkedAt": _now_iso(),
    }
    try:
        rel = latest_release(channel)
    except Exception as e:
        return dict(base, ok=False, updateAvailable=False, latest="",
                    error=_update_error_text(e))
    newer = bool(rel["tag"]) and _release_is_newer(
        rel["tag"], current, current_status, rel.get("buildSeq"))
    # The release's notes travel with the check result (build .70,
    # issue #47): the Updater panel reads them before offering the
    # install, and Apply never fetches them again. Source of truth
    # is the release body; an empty body falls back to the matching
    # CHANGELOG section, and notesSource says which one answered,
    # so the panel can label the fallback honestly.
    notes = rel["notes"]
    notes_source = "release" if notes.strip() else ""
    if newer and not notes_source:
        fallback = changelog_section_for(rel["tag"])
        if fallback:
            notes = fallback
            notes_source = "changelog"
    # The running version's own release page rides the same check
    # (build .83): the panel's "About this release" dropdown shows
    # everything that page offers without a second round trip.
    cur_rel = current_release(cfg)
    if cur_rel is not None and not cur_rel["notes"].strip():
        fb = changelog_section_for(cur_rel["tag"])
        if fb:
            cur_rel = dict(cur_rel, notes=fb, notesSource="changelog")
    return dict(
        base, ok=True, updateAvailable=newer,
        latest=rel["tag"], releaseName=rel["name"],
        releaseUrl=rel["url"], publishedAt=rel["publishedAt"],
        prerelease=rel["prerelease"],
        buildSeq=rel.get("buildSeq"), currentBuildSeq=SHIPPED_BUILD_SEQ,
        notes=notes, notesSource=notes_source, asset=rel["asset"],
        currentRelease=cur_rel,
    )


def _version_label(number, status):
    # The footer's version-label form: "v" + number + the status
    # letter (e for edge, b for beta, none for a full release) —
    # the form the Updater's confirmation line uses for both
    # sides of "Updated v0.1.6a → v0.1.7" (the from-side's "a" is
    # a legacy tag from before the sync; it is read, never made).
    return "v" + str(number) + (
        "e" if status in ("edge", "alpha") else
        "b" if status == "beta" else "")


def _tag_parts(tag):
    # A release tag in the label's shape — "v0.1.8e", "v0.1.7",
    # "v0.1.8-edge" — split into (number, status). The retired
    # alpha forms ("v0.1.6a", "-alpha") parse as edge: they were
    # pre-releases then and are pre-releases now.
    text = str(tag or "").strip()
    if text.lower().startswith("v"):
        text = text[1:]
    status = "stable"
    low = text.lower()
    for word in ("edge", "beta", "alpha"):
        if low.endswith("-" + word) or low.endswith(" " + word):
            status = "edge" if word == "alpha" else word
            text = text[: -(len(word) + 1)]
            low = text.lower()
            break
    if (status == "stable" and len(text) > 1
            and text[-1].lower() in ("e", "a", "b") and text[-2].isdigit()):
        status = "beta" if text[-1].lower() == "b" else "edge"
        text = text[:-1]
    return text, status


def _write_update_result(rec):
    try:
        # Build 6: every write carries a freshness stamp, so a
        # reader can always tell a live apply from an orphaned
        # one (see the apply watchdog below).
        rec["updatedAt"] = _now_iso()
        rec["updatedEpoch"] = time.time()
        os.makedirs(os.path.dirname(UPDATE_RESULT_PATH), exist_ok=True)
        tmp = UPDATE_RESULT_PATH + ".tmp"
        with open(tmp, "w") as f:
            json.dump(rec, f)
        os.replace(tmp, UPDATE_RESULT_PATH)
    except Exception:
        pass


def _apply_watchdog(rec):
    # The apply watchdog (Build 6): the backstop for the one
    # death the detached helper cannot record itself — killed
    # outright (SIGKILL, the kernel OOM killer, power loss)
    # before it could write again, like the silent death the
    # Build 4 round-trip gate found, where the record froze at
    # working/installing for a quarter of an hour and nothing
    # ever closed it. When a record still claims "working" but
    # its helper process is gone and its last write is stale,
    # the record is closed HERE, by whichever bridge serves it
    # next: state failed, the step it died on kept, and the
    # error quoting the helper step log's last line (the last
    # step the helper provably completed). A helper that is
    # still alive is never touched: the install step can
    # legitimately run silent for many minutes while the
    # installer rebuilds, and only a dead pid plus a stale
    # record convicts. Records with no helper pid (written
    # before pids were recorded) are judged on staleness alone,
    # against a threshold longer than any silent step.
    try:
        if rec.get("state") != "working":
            return rec
        try:
            stale = time.time() - os.path.getmtime(UPDATE_RESULT_PATH)
        except Exception:
            return rec
        pid = rec.get("helperPid")
        alive = None
        if pid:
            try:
                alive = pid_alive(int(pid))
            except Exception:
                alive = None
        if alive:
            return rec
        if alive is False and stale <= 30:
            return rec
        if alive is None and stale <= 360:
            return rec
        step = rec.get("step") or "installing"
        last = ""
        trace_path = rec.get("traceLog") or ""
        if trace_path:
            try:
                with open(trace_path, "rb") as f:
                    lines = f.read().decode(
                        "utf-8", "replace").strip().splitlines()
                if lines:
                    last = lines[-1].strip()
            except Exception:
                last = ""
        rec["state"] = "failed"
        rec["failedAt"] = _now_iso()
        rec["watchdog"] = True
        rec["error"] = (
            "The apply helper stopped at the " + step + " step "
            "and never finished this record — the apply watchdog "
            "closed it, because the helper process is gone and "
            "the record went stale. Nothing is confirmed updated."
            + (" Its step log ends with: " + last if last else "")
            + " The staged package is still in place — the "
            "manual fallback below finishes the job by hand.")
        _write_update_result(rec)
        return rec
    except Exception:
        return rec


def update_result_payload():
    # The apply's live record (config/update-result.json): the
    # step it is on while it runs, and afterwards either the
    # confirmed outcome or the failure + fallback. No record yet
    # reads as state "none". A record orphaned by a helper that
    # died outright is closed by the apply watchdog on its way
    # out (Build 6) — never left claiming "working" forever.
    try:
        with open(UPDATE_RESULT_PATH) as f:
            rec = json.load(f)
        if isinstance(rec, dict):
            out = dict(_apply_watchdog(rec), ok=True)
            # Build 3 (#175): the failure card pairs its fix with
            # the reading — a "See what happened" command beside
            # the manual fallback, derived HERE, at the one read
            # every record passes through, so a record written by
            # the helper, by the bridge, or closed by the watchdog
            # all carry it. The staged folder's parent is the
            # tag's folder under updates/, where the apply's two
            # logs live: the installer's own output and the
            # helper's step log.
            staged = str(out.get("stagedDir") or "")
            if staged:
                tag_dir = os.path.dirname(staged)
                out["diagnoseCommand"] = (
                    "cat " + os.path.join(tag_dir, "apply-install.log")
                    + "; cat " + os.path.join(tag_dir, "apply-trace.log"))
            return out
    except Exception:
        pass
    return {"ok": True, "state": "none"}


def _unwrap_package(data):
    # The download validator's front door (build .56): returns
    # (payload, note) — payload is the zip bytes to verify, note
    # says how they arrived ("" for a plain zip, "gzip" when a
    # gzip wrapper was unwrapped). When neither the raw payload
    # nor its unwrapped form can be a zip, payload is None and
    # note names the shape that failed ("gzip-broken",
    # "gzip-not-zip"); a plain non-zip payload comes back as-is
    # and fails the zip check with the classic wording, exactly
    # as before.
    if data[:2] == bytes([0x1F, 0x8B]):
        try:
            inner = gzip.decompress(data)
        except Exception:
            return None, "gzip-broken"
        if inner[:2] == b"PK":
            return inner, "gzip"
        return None, "gzip-not-zip"
    return data, ""


# The detached apply helper, written by the bridge into
# updates/<tag>/apply-helper.py at apply time and spawned in its
# own session so it outlives the bridge: it runs the staged
# installer (never --fresh — persistence is the installer's own
# stash-and-restore), relaunches the app through start.sh, and
# finishes the update-result record the panel reads. It is a
# standalone stdlib script: everything it needs arrives in the
# job file beside it. Since Build 6 it also keeps the step log
# (apply-trace.log) described in its own header: the Build 4
# round-trip gate caught the old helper dying traceless
# between the installer's "Done." and the relaunch — no log,
# no record update, no relaunch — and the repair is structural
# (trace from the first instruction, a top-level guard, signal
# handlers, a heartbeat, and the bridge-side apply watchdog in
# update_result_payload above), not a patch on one suspect
# line.
APPLY_HELPER_PY = '''#!/usr/bin/env python3
# Apex updater apply helper — written by bridge.py into
# updates/<tag>/ and run detached (its own session), so it
# outlives the bridge it replaces. It runs the staged installer
# (never --fresh), relaunches the app through start.sh, and
# records the outcome in config/update-result.json — the record
# the Updater panel reads its confirmation (or its failure and
# fallback) from. Stdlib only, like the bridge.
#
# Build 6 — the helper can no longer die traceless. The Build 4
# round-trip gate caught it dying silently: the installer ran
# through to "Done.", then nothing — apply.log 0 bytes, no
# relaunch log, the record frozen at working/installing, no
# bridge relaunched. The old helper's only diary was the record
# itself, it wrote the record only at step boundaries, it had
# no top-level guard, no signal handlers, and it printed
# nothing — so any death between two writes (a signal above
# all: an uncaught Python exception would at least have left a
# traceback in apply.log, and apply.log was empty) left no
# evidence at all. Now, from its first instruction, the helper
# keeps a step log — apply-trace.log beside the job file, its
# path carried on the apply record — and writes every step
# boundary, a 30-second heartbeat while it works, every signal
# it is handed, and every exception into that log and onto the
# record (state/step/updatedAt) as it happens. A top-level
# guard turns any Python-level failure into a failed record
# naming the step and the reason before exiting, and mirrors
# the traceback to stderr so apply.log is never 0 bytes on a
# death words could have described. Signal handlers do the
# same for SIGTERM, SIGINT and SIGHUP. The one death no
# in-process guard can survive — SIGKILL, the kernel OOM
# killer, power loss — still leaves the step log ending at the
# last completed step, and the bridge's apply watchdog closes
# a record whose helper is gone as failed at that step,
# quoting the log's last line, instead of leaving it frozen
# at working/installing forever.
#
# v0.4.3e (#237) — the stage comes home. The v0.4.2e field
# pull named the remaining updater wounds: the installer's
# own output and the staged package both lived in the stage
# folder inside the tree the installer's swap parked, so
# the paths the failure surface printed (apply-install.log
# for the diagnose command, the package folder for the
# manual fallback) did not exist in the live tree — and,
# found by the same bake, the installer itself died after
# the swap when it recomputed its own folder's path (fixed
# in install.sh). This helper now holds a copy of the staged
# package beside the install root while the installer runs,
# keeps the install log's handle open across the swap, and
# — before any verdict or record close — brings the stage
# home: the log written to the live path the diagnose
# command names, the package restored to the folder the
# fallback command points at. It also fails the record on
# the installer's owed-but-missing line: a shell build that
# was owed (the sources changed) and produced no window is
# a missing window, not the designed browser fallback.
import json
import os
import re
import shutil
import signal
import socket
import subprocess
import sys
import threading
import time
import traceback
import urllib.request

STATE = {"step": "starting", "job": None, "env": None,
         "trace_path": None, "result_path": None,
         "stop_beat": False}


def _stamp():
    return time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())


def trace(event, detail=""):
    # The step log: append-only, flushed and fsync'd per line,
    # and never allowed to raise — a helper that dies because
    # its own diary failed would be the old bug in a new coat.
    path = STATE.get("trace_path")
    if not path:
        return
    try:
        line = _stamp() + " [" + str(STATE.get("step")) + "] " + event
        if detail:
            line += " — " + str(detail)
        # The stage folder rides inside the tree the installer
        # swaps: after a same-home install it sits parked with
        # the previous tree, and the diary must not go silent
        # at exactly the steps (relaunch, confirmation) it
        # exists to record. Recreate the folder and write on.
        _tdir = os.path.dirname(path)
        if _tdir:
            os.makedirs(_tdir, exist_ok=True)
        with open(path, "a") as f:
            f.write(line + "\\n")
            f.flush()
            os.fsync(f.fileno())
    except Exception:
        pass


def write_result(patch):
    # The apply record: read-modify-write like always, now
    # stamped with the current step and a freshness time on
    # EVERY write, so the panel — and the bridge's apply
    # watchdog — can always tell a live helper from a dead
    # one. Never raises: a failed record write is traced and
    # retried once, and the step log still carries what the
    # record could not.
    job = STATE.get("job") or {}
    result_path = STATE.get("result_path") or job.get("resultPath")
    if not result_path:
        return
    patch = dict(patch)
    if "step" in patch:
        STATE["step"] = patch["step"]
    patch.setdefault("step", STATE.get("step"))
    patch["updatedAt"] = _stamp()
    patch["updatedEpoch"] = time.time()
    patch["helperStep"] = STATE.get("step")
    if job.get("traceLog"):
        patch["traceLog"] = job["traceLog"]
    for attempt in (1, 2):
        try:
            rec = {}
            try:
                with open(result_path) as f:
                    rec = json.load(f)
                if not isinstance(rec, dict):
                    rec = {}
            except Exception:
                rec = {}
            rec.update(patch)
            # v0.3.0e Build 2 (#208): a record must never argue
            # with itself. The merge above only ever ADDS keys,
            # so a previous attempt's error/failedAt/tailFailure
            # survived into a later attempt that recovered to done — the
            # field record for the v0.3.0e Build 1 pull read
            # state=done while still carrying the earlier wedge
            # error and its failedAt stamp. A verdict owns its
            # record: done clears the failure fields outright,
            # and a fresh working attempt clears a previous
            # attempt's, so what the panel reads is always THIS
            # attempt's truth.
            if patch.get("state") == "done":
                rec.pop("error", None)
                rec.pop("failedAt", None)
                rec.pop("tailFailure", None)
            elif patch.get("state") == "working":
                rec.pop("error", None)
                rec.pop("failedAt", None)
                rec.pop("tailFailure", None)
                rec.pop("tailFailure", None)
            tmp = result_path + ".tmp"
            with open(tmp, "w") as f:
                json.dump(rec, f)
            os.replace(tmp, result_path)
            trace("record",
                  "state=" + str(rec.get("state"))
                  + " step=" + str(rec.get("step")))
            return
        except Exception as e:
            trace("record-write-failed",
                  "attempt " + str(attempt) + ": " + str(e))


def fail(step, error):
    STATE["step"] = step
    job = STATE.get("job") or {}
    write_result({
        "state": "failed", "step": step, "error": error,
        "stagedDir": job.get("pkgDir", ""),
        "command": job.get("command", ""),
        "failedAt": _stamp(),
    })


def _heartbeat():
    # While the helper lives it beats every 30 seconds into the
    # step log, naming the step it is on. A log whose beats stop
    # mid-step is a killed helper; a log still beating at one
    # step is a blocked one — the field can finally tell those
    # two deaths apart, which the frozen Build 4 record could
    # not.
    while not STATE.get("stop_beat"):
        time.sleep(30)
        if STATE.get("stop_beat"):
            break
        trace("heartbeat", "still working")


def _on_signal(signum, frame):
    # SIGTERM / SIGINT / SIGHUP get words before the exit: the
    # step log and the record both name the signal and the step
    # it interrupted, so a helper stopped from outside is a
    # recorded failure, never a silent freeze. (SIGKILL cannot
    # be handled by any process — the step log's last line and
    # the bridge's apply watchdog cover that death.)
    name = "signal " + str(signum)
    try:
        name = signal.Signals(signum).name
    except Exception:
        pass
    trace("signal", name + " received — recording the failure and exiting")
    job = STATE.get("job") or {}
    try:
        write_result({
            "state": "failed", "step": STATE.get("step"),
            "error": (
                "The apply helper was stopped by " + name
                + " at the " + str(STATE.get("step")) + " step, "
                "so the update is not confirmed. Its step log "
                "(apply-trace.log, beside the staged package) "
                "shows the last step it completed. The staged "
                "package is still in place — the manual fallback "
                "below finishes the job by hand."),
            "stagedDir": job.get("pkgDir", ""),
            "command": job.get("command", ""),
            "failedAt": _stamp(),
        })
    except Exception:
        pass
    os._exit(128 + signum)


def version_tuple(raw):
    text = str(raw or "").strip().lower()
    if text.startswith("v"):
        text = text[1:]
    text = text.split("-")[0].split("+")[0]
    parts = []
    for piece in text.split("."):
        digits = ""
        for ch in piece:
            if ch.isdigit():
                digits += ch
            else:
                break
        parts.append(int(digits) if digits else 0)
    return tuple(parts) or (0,)


def tuples_equal(a, b):
    width = max(len(a), len(b))
    return a + (0,) * (width - len(a)) == b + (0,) * (width - len(b))


def read_installed_version(root):
    # The VERSION file at the install root, parsed the way the
    # bridge parses it: (number, status).
    try:
        with open(os.path.join(root, "VERSION")) as f:
            text = f.read().strip()
    except Exception:
        return "", "stable"
    if not text:
        return "", "stable"
    raw = text.splitlines()[0].strip()
    status = "stable"
    low = raw.lower()
    for word in ("edge", "beta", "alpha"):
        if low.endswith(" " + word) or low.endswith("-" + word):
            status = "edge" if word == "alpha" else word
            raw = raw[: -(len(word) + 1)]
            break
    parts = raw.split()
    token = parts[0] if parts else ""
    for word in ("edge", "alpha", "beta", "stable"):
        if token.lower().endswith("-" + word):
            token = token[: -(len(word) + 1)]
            break
    return token, status


def read_installed_build_seq(root):
    # The Edge build sequence baked into the bridge.py at the
    # install root (Build 4): the on-disk half of the identity a
    # same-version refresh has to prove, beside the VERSION
    # file's number — which a refresh deliberately does not
    # change. None when the file or the sequence cannot be read
    # (a bridge from before build sequences existed).
    try:
        with open(os.path.join(root, "bridge.py"), "rb") as f:
            m = re.search(
                rb"SHIPPED_BUILD_SEQ\\s*=\\s*([0-9]+)", f.read())
        if m:
            return int(m.group(1))
    except Exception:
        pass
    return None


def _shell_window_state(root):
    # v0.4.2e (#233): the apply's own verdict on the installed
    # standalone window, read from DISK after the installer
    # exits — never from the installer's exit code. Returns
    # (state, reason):
    #   "ok"      — the binary's build stamp covers the
    #               installed shell sources, or no binary
    #               stands beside start.sh at all (the
    #               designed browser-fallback installs retire
    #               theirs, and the launcher falls back
    #               without complaint).
    #   "stale"   — a binary stands there whose stamp does not
    #               cover the installed sources: exactly the
    #               state start.sh refuses at the next launch,
    #               and the state v0.4.1e's pull produced while
    #               reporting success.
    #   "unknown" — the sources cannot be hashed here, so
    #               nothing can be proven either way. Named in
    #               the trace, never silently passed, never
    #               failed on suspicion.
    # The hash is the installer's own pipeline (shell_sources_
    # hash in install.sh): sha256sum of the source files, the
    # listing hashed again — replicated call for call so the
    # two can never drift into different answers.
    shell_dir = os.path.join(root, "shell")
    binary = os.path.join(root, "navi-apex-shell")
    if not os.path.isfile(binary):
        return "ok", ""
    files = ["Cargo.toml", "build.rs", "src/main.rs",
             "tauri.conf.json"]
    if os.path.isfile(os.path.join(shell_dir, "icons", "icon.png")):
        files.append("icons/icon.png")
    files.append("README.md")
    for f in files:
        if not os.path.isfile(os.path.join(shell_dir, f)):
            return "unknown", ("the installed shell sources are "
                               "incomplete (" + f + " is missing), "
                               "so the window's build stamp cannot "
                               "be verified")
    try:
        p1 = subprocess.run(["sha256sum"] + files, cwd=shell_dir,
                            capture_output=True, text=True,
                            timeout=60)
        if p1.returncode != 0 or not p1.stdout:
            return "unknown", ("sha256sum could not hash the "
                               "installed shell sources")
        p2 = subprocess.run(["sha256sum"], input=p1.stdout,
                            capture_output=True, text=True,
                            timeout=60)
        parts = (p2.stdout or "").split()
        digest = parts[0] if p2.returncode == 0 and parts else ""
    except Exception as e:
        return "unknown", ("the shell sources could not be "
                           "hashed here: " + str(e))
    if not digest:
        return "unknown", ("the shell sources could not be "
                           "hashed here")
    stamp = ""
    try:
        with open(os.path.join(root, "navi-apex-shell.stamp")) as f:
            stamp = f.read().strip()
    except Exception:
        stamp = ""
    if not stamp:
        return "stale", ("the window binary beside start.sh "
                         "carries no build stamp "
                         "(navi-apex-shell.stamp is missing), so "
                         "the launcher will refuse it as "
                         "unverifiable and open the browser "
                         "instead")
    if stamp != digest:
        return "stale", ("the window binary beside start.sh was "
                         "built from older shell sources — its "
                         "build stamp does not cover the shell "
                         "sources this update installed, so the "
                         "launcher will refuse it and open the "
                         "browser instead")
    return "ok", ""


def _relaunch_systemd_owned(job):
    # v0.6.7e (#305): does systemd own this install's bridge?
    # The job's startup mode is the install's own recorded
    # answer: an "on-boot" install registered systemd user
    # units (navi-llamafile.service, navi-bridge.service) and
    # the installer started them itself. On such a seat a
    # start.sh spawn can only adopt the unit's bridge and then
    # hold the foreground polling it — one immortal start.sh
    # per Apply, in a transient scope, lying about who
    # supervises the bridge.
    return bool(job) and job.get("startup") == "on-boot"


def _install_headless(job, root):
    # v0.4.6e (#185): is this a headless install? Returns
    # (headless, source). The determination mirrors the
    # installer's own precedence (apex_headless_decide in
    # install.sh), from the witnesses an apply actually has:
    #   1. the job itself, when a bridge that knows the field
    #      names it ("headless": true);
    #   2. the installed config's recorded determination —
    #      install.sh stamps config "headless" beside the
    #      startup choice, so after the installer runs this is
    #      the installer's own verdict for THIS apply, and
    #      before it runs it is the previous install's carried
    #      choice (the same carry the startup mode rides);
    #   3. --startup boot implies headless: boot services are
    #      the headless shape;
    #   4. auto-detection from this process's environment: no
    #      DISPLAY and no WAYLAND_DISPLAY and no graphical
    #      session type.
    # A recorded false never vetoes auto-detection, exactly as
    # in the installer: it records a past headed install, not
    # a display on today's seat.
    if job.get("headless") is True:
        return True, "the apply job names this install headless"
    try:
        with open(os.path.join(root, "config",
                               "navi.config.json")) as f:
            cfg = json.load(f)
        if isinstance(cfg, dict) and cfg.get("headless") is True:
            return True, ("the recorded determination in the "
                          "installed config (config headless: true)")
    except Exception:
        pass
    if job.get("startup") == "on-boot":
        return True, "--startup boot implies headless (#185)"
    if not os.environ.get("DISPLAY") \\
            and not os.environ.get("WAYLAND_DISPLAY") \\
            and os.environ.get("XDG_SESSION_TYPE", "") \\
            not in ("x11", "wayland", "mir"):
        return True, ("auto-detected — no DISPLAY, no "
                      "WAYLAND_DISPLAY and no graphical session "
                      "in the apply's environment")
    return False, "a display is present (or a graphical session is recorded)"


def _pid_alive(pid):
    try:
        os.kill(pid, 0)
        return True
    except Exception:
        return False


def _pid_gone(pid):
    # Gone = no such process, or a zombie: a zombie holds no
    # sockets (the kernel closed them at death) and its parent
    # reaps it in its own time.
    if not pid:
        return True
    if not _pid_alive(pid):
        return True
    try:
        with open("/proc/" + str(pid) + "/stat", "rb") as f:
            parts = f.read().rsplit(b")", 1)
        state = parts[1].split()[0].decode("utf-8", "replace") if len(parts) > 1 else ""
        return state == "Z"
    except Exception:
        return False


def _pid_cmdline(pid):
    try:
        with open("/proc/" + str(pid) + "/cmdline", "rb") as f:
            raw = f.read()
        return " ".join(part.decode("utf-8", "replace") for part in raw.split(bytes([0]))).strip()
    except Exception:
        return ""


def _pid_cwd(pid):
    try:
        return os.path.realpath(os.readlink("/proc/" + str(pid) + "/cwd"))
    except Exception:
        return ""


def _port_holder_pid(port):
    # The PID holding the LISTEN socket on the port, resolved from
    # /proc alone — names a holder that answers nothing, where a
    # status probe can only shrug.
    inodes = set()
    for path in ("/proc/net/tcp", "/proc/net/tcp6"):
        try:
            with open(path) as f:
                rows = f.read().splitlines()[1:]
        except Exception:
            continue
        for row in rows:
            parts = row.split()
            if len(parts) > 9 and parts[3] == "0A":
                try:
                    if int(parts[1].rsplit(":", 1)[1], 16) == port:
                        inodes.add(parts[9])
                except Exception:
                    pass
    if not inodes:
        return None
    targets = set("socket:[" + ino + "]" for ino in inodes)
    try:
        names = os.listdir("/proc")
    except Exception:
        return None
    for name in names:
        if not name.isdigit():
            continue
        try:
            fds = os.listdir("/proc/" + name + "/fd")
        except Exception:
            continue
        for fd in fds:
            try:
                if os.readlink("/proc/" + name + "/fd/" + fd) in targets:
                    return int(name)
            except Exception:
                pass
    return None


def _port_answers_expected(port, tgt_seq):
    # True when the bridge answering on the port already reports
    # the target identity — the case where a relaunch spawn
    # failure must NOT fail the apply: the installer's own
    # relaunch (build .91) already brought the new bridge up.
    try:
        req = urllib.request.Request(
            "http://127.0.0.1:" + str(port) + "/api/status",
            headers={"User-Agent": "navi-update-helper"})
        with urllib.request.urlopen(req, timeout=3) as resp:
            seen = json.loads(resp.read().decode("utf-8", "replace"))
    except Exception:
        return False
    if not isinstance(seen, dict) or seen.get("app") != "Navi: Apex":
        return False
    return tgt_seq is None or seen.get("buildSeq") == tgt_seq


def _holder_remedy(port):
    # The failure-text remedy for a held port (build .91): name
    # the holder and its exact kill command when the socket can
    # resolve it — never "launch the app again" against a wedged
    # port, the advice that stranded the field update this build
    # answers.
    holder = _port_holder_pid(port)
    if holder:
        cmd = _pid_cmdline(holder)
        return ("The port is held by process " + str(holder)
                + (" (" + cmd + ")" if cmd else "")
                + ": end it with kill -9 " + str(holder)
                + ", then run the update again — launching the app "
                "again cannot work while that process holds the port.")
    return ("If a process is holding the port, find it with lsof -i :"
            + str(port) + " and end it, then run the update again; "
            "otherwise launch the app again from its menu entry.")


def _find_install_bridge(root):
    # v0.3.0e Build 2 (#208): find THIS install's bridge by the
    # installer's own proof discipline — the pid file the bridge
    # itself writes (bridge.pid), verified against the process
    # table, and a working-directory scan for a bridge.py whose
    # cwd is the install home. Whoever supervises the bridge —
    # the systemd user unit or the desktop app (start.sh's
    # supervise loop) — the process it runs is the same shape:
    # bridge.py, cwd = install home. Returns a proven pid or
    # None. A pid counts only when BOTH proofs hold (bridge.py
    # in its command line AND its cwd the install root): the
    # same bar the socket path has always used.
    home = os.path.realpath(root)
    candidates = []
    try:
        with open(os.path.join(root, "bridge.pid")) as f:
            pid = int((f.read() or "").strip() or 0)
        if pid:
            candidates.append(pid)
    except Exception:
        pass
    try:
        for name in os.listdir("/proc"):
            if name.isdigit():
                candidates.append(int(name))
    except Exception:
        pass
    seen = set()
    for pid in candidates:
        if pid in seen or pid == os.getpid():
            continue
        seen.add(pid)
        if _pid_gone(pid):
            continue
        if "bridge.py" in _pid_cmdline(pid) and _pid_cwd(pid) == home:
            return pid
    return None


def _stop_proven_bridge(holder, port, desc):
    # The escalation both identification paths share: SIGTERM
    # (the bridge's polite exit — the installer's stop does the
    # same), wait for the process to be gone AND the port free,
    # then SIGKILL and wait again. Returns True when the port
    # is clear.
    trace("wedge-clear",
          "wedged bridge " + desc + " — escalating: SIGTERM, then SIGKILL if it holds")
    try:
        os.kill(holder, signal.SIGTERM)
    except Exception:
        pass
    deadline = time.time() + 15
    while time.time() < deadline:
        if _pid_gone(holder) and _port_holder_pid(port) is None:
            trace("wedge-clear", "holder gone after SIGTERM; the port is free")
            return True
        time.sleep(0.5)
    try:
        os.kill(holder, signal.SIGKILL)
    except Exception:
        pass
    deadline = time.time() + 10
    while time.time() < deadline:
        if _pid_gone(holder) and _port_holder_pid(port) is None:
            trace("wedge-clear", "holder gone after SIGKILL; the port is free")
            return True
        time.sleep(0.5)
    return False


def clear_wedged_bridge(root, port, tgt_seq):
    # Build .91 (issue #91): before relaunching, prove the OLD
    # bridge is actually gone — not merely that its pid file is.
    # A bridge can still hold the port while answering nothing
    # (frozen, stopped, stuck mid-work), and every relaunch
    # against it is a no-op: the new bridge cannot bind, and the
    # confirmation that follows can only report failure with a
    # remedy that cannot work. If the target build already
    # answers, there is nothing to clear; if the port is free,
    # likewise. A holder is signalled ONLY when it is provably
    # this install's bridge — bridge.py in its command line AND
    # its working directory the install root; anything else is
    # reported, never touched. Escalation mirrors the installer:
    # SIGTERM, wait for death AND a free port, SIGKILL, wait
    # again. Returns (ok, holder_pid, holder_desc).
    #
    # v0.3.0e Build 2 (#208): identification no longer rests on
    # the socket alone. The v0.3.0e Build 1 field pull wedged
    # exactly here: the port answered, the socket named no
    # readable process, and the old text declared the holder
    # unidentifiable and told the operator to kill it by hand —
    # while the holder was this install's own bridge, run by
    # the desktop app, findable the whole time by the pid file
    # and the working-directory proof the installer itself
    # uses. When the socket cannot name the holder, the helper
    # now looks the bridge up the installer's way before it
    # ever says "unidentifiable" — and when it truly cannot
    # identify anything, the failure names every proof it ran.
    if _port_answers_expected(port, tgt_seq):
        return (True, None, "")
    holder = _port_holder_pid(port)
    if holder is None:
        try:
            sock = socket.create_connection(("127.0.0.1", port), timeout=2)
            sock.close()
        except Exception:
            return (True, None, "")
        found = _find_install_bridge(root)
        if found is None:
            return (False, None,
                    "something is listening on the port but its process "
                    "could not be identified — the socket named no readable "
                    "process, the install's pid file (bridge.pid) named no "
                    "live bridge, and no bridge.py process runs from this "
                    "install home")
        holder = found
        cmd = _pid_cmdline(holder)
        desc = ("process " + str(holder) + ((" (" + cmd + ")") if cmd else "")
                + " — the install's own bridge, found by its pid file / "
                "working directory after the socket named no process")
        if _stop_proven_bridge(holder, port, desc):
            return (True, holder, desc)
        return (False, holder, desc)
    cmd = _pid_cmdline(holder)
    desc = "process " + str(holder) + ((" (" + cmd + ")") if cmd else "")
    owned = ("bridge.py" in cmd) and (_pid_cwd(holder) == os.path.realpath(root))
    if not owned:
        return (False, holder, desc + " — not this install's bridge, so it was not touched")
    if _stop_proven_bridge(holder, port, desc):
        return (True, holder, desc)
    return (False, holder, desc)


def _install_log_last_step(log_path):
    # Issue #104: the installer's own step banners ("==> ...")
    # name its steps; the last banner in the log is the step the
    # run had reached when it failed — the tail step a nonzero
    # exit is reported with when the build itself landed.
    #
    # v0.3.0e Build 2 (#208): the step is NEVER "an unnamed
    # step" again — the v0.3.0e Build 1 field record carried
    # exactly that phrase as its tailFailure, a name that names
    # nothing. When no banner survives, the log's last ERROR
    # line names the failure in the installer's own words; when
    # even that is absent, the fallback names the phase plainly
    # (a post-placement closing step) instead of shrugging.
    try:
        with open(log_path, "rb") as f:
            text = f.read().decode("utf-8", "replace")
    except Exception:
        return "the installer's closing steps (its log could not be read)"
    last = ""
    last_error = ""
    for line in text.splitlines():
        line = line.strip()
        if line.startswith("==>") or line.startswith("== "):
            last = line.lstrip("=> ").strip()
        elif line.startswith("ERROR") or " — FAILED" in line:
            last_error = line
    if last:
        return last
    if last_error:
        return "the step that reported: " + last_error
    return "the installer's closing steps (its log names no step banner for the failure)"


def _stage_holding_dir(job):
    # The apply's insurance folder: a sibling of the install
    # root, in the installer's own sibling idiom (.apex-stage,
    # .apex-prev) — ground the installer's swap never renames.
    root = str(job.get("root") or "")
    tag = str(job.get("tagDir") or "apply")
    if not root:
        return ""
    return os.path.join(
        os.path.dirname(os.path.abspath(root)),
        ".apex-apply-stage", tag)


def _recover_stage_artifacts(job, log_handle):
    # v0.4.3e (#237): the stage comes home. The job's paths —
    # the install log, the staged package — name the stage
    # folder inside the tree the installer replaces, and the
    # swap parks that tree beside its successor (.apex-prev):
    # in the field (v0.4.2e's pull on neondrip) the parked
    # tree took the install log and the package with it, so
    # every path the failure surface printed — the diagnose
    # command's apply-install.log, the manual fallback's
    # package folder — was a path the apply had NOT left
    # behind, and the operator's own fallback cd failed with
    # "No such file or directory". The trace and relaunch
    # logs survive the swap only because their writers
    # recreate the folder; the install log and the package
    # had no such rescue. Now they do, here, before any
    # record closes:
    #   * the install log is read back through this helper's
    #     own still-open handle (an open handle survives the
    #     rename AND the installer's end-of-run deletion of
    #     the previous tree) and written to the live path the
    #     diagnose command names;
    #   * the staged package is restored to the live folder
    #     the fallback command names — from the parked tree
    #     when it stands, else from the holding copy this
    #     helper made before the installer ran.
    # A path that already stands (a no-swap apply, a parked-
    # in-place migration) is left exactly as it lies.
    # Best-effort throughout: a rescue that fails is traced,
    # never fatal — the apply's verdict must not hinge on it.
    try:
        install_log = str(job.get("installLog") or "")
        if install_log and not os.path.isfile(install_log) \\
                and log_handle is not None:
            try:
                log_handle.flush()
                log_handle.seek(0)
                data = log_handle.read()
            except Exception:
                data = b""
            if data:
                _idir = os.path.dirname(install_log)
                if _idir:
                    os.makedirs(_idir, exist_ok=True)
                with open(install_log, "wb") as f:
                    f.write(data)
                trace("stage-recovered",
                      "the install log was brought home to "
                      + install_log + " (" + str(len(data))
                      + " bytes, read back through the helper's "
                      "own handle — the swap had parked the "
                      "original with the previous tree)")
        pkg_dir = str(job.get("pkgDir") or "")
        if pkg_dir and not os.path.isdir(pkg_dir):
            src = ""
            orig_root = os.path.abspath(str(job.get("root") or ""))
            if orig_root and os.path.abspath(pkg_dir).startswith(
                    orig_root + os.sep):
                parked = os.path.join(
                    os.path.dirname(orig_root), ".apex-prev") \\
                    + os.path.abspath(pkg_dir)[len(orig_root):]
                if os.path.isdir(parked):
                    src = parked
            if not src:
                held = os.path.join(
                    _stage_holding_dir(job), "package")
                if held and os.path.isdir(held):
                    src = held
            if src:
                _pdir = os.path.dirname(pkg_dir)
                if _pdir:
                    os.makedirs(_pdir, exist_ok=True)
                shutil.copytree(src, pkg_dir)
                trace("stage-recovered",
                      "the staged package was restored to "
                      + pkg_dir + " from " + src
                      + " — the manual fallback's folder stands "
                      "where the fallback command points")
            else:
                trace("stage-recovery-missed",
                      "no surviving copy of the staged package "
                      "was found to restore to " + pkg_dir)
    except Exception as e:
        trace("stage-recovery-failed", str(e))
    # The holding copy has served either way; a finished
    # apply leaves no litter beside the install root.
    try:
        held_dir = _stage_holding_dir(job)
        if held_dir and os.path.isdir(held_dir):
            shutil.rmtree(held_dir, ignore_errors=True)
            try:
                os.rmdir(os.path.dirname(held_dir))
            except Exception:
                pass
    except Exception:
        pass


def version_label(number, status):
    return "v" + number + (
        "e" if status in ("edge", "alpha") else
        "b" if status == "beta" else "")

