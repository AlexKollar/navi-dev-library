# bridge.py — part 11 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 41544–44872 of bridge.py.
Top-level symbols in this part: Handler.


class Handler(BaseHTTPRequestHandler):
    def _send(self, code, payload, ctype="application/json"):
        body = payload if isinstance(payload, bytes) else json.dumps(payload).encode()
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        # Nothing this bridge serves may come out of a browser cache.
        # The dashboard lives at the same localhost:8675 the operator's
        # own earlier UI used, so a cached page from that era would
        # otherwise keep answering in the bridge's place — the browser
        # showing an old UI while a current bridge serves underneath.
        self.send_header("Cache-Control", "no-store, no-cache, must-revalidate")
        self.send_header("Pragma", "no-cache")
        self.send_header("Expires", "0")
        self.end_headers()
        self.wfile.write(body)

    def _body(self):
        length = int(self.headers.get("Content-Length") or 0)
        return json.loads(self.rfile.read(length) or b"{}")

    def _client_is_loopback(self):
        try:
            host = str(self.client_address[0] or "")
        except Exception:
            return False
        return host == "::1" or host.startswith("127.")

    def _public_gate_blocks(self):
        # The gate exists only while the bridge is actually bound
        # publicly (public mode was ON at launch). Loopback callers
        # are never gated. Everyone else is judged by the device
        # registry (Build 2, #173): a trusted device's own token
        # opens; the shared-code cookie opens only while no device
        # has ever been approved; everything else is blocked.
        if ACTIVE_BIND_HOST != "0.0.0.0":
            return False
        if self._client_is_loopback():
            return False
        cfg = load_config()
        if not public_settings(cfg)["access_code"]:
            return True
        ok, dev, _legacy = door_gate_acceptance(cfg, door_cookie_value(self.headers.get("Cookie")))
        if ok and dev is not None:
            door_touch_device(cfg, dev)
        return not ok

    def _gate_pending_poll(self):
        # The waiting device's poll on the bridge's own door (the
        # DHD proxy answers its own; same registry, same words).
        q = urllib.parse.parse_qs(urllib.parse.urlsplit(self.path).query)
        pid = (q.get("id") or [""])[0]
        status, dev = door_pending_poll(load_config(), pid)
        body = json.dumps({"status": status}).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        if status == "approved" and dev is not None and dev.get("token"):
            self.send_header("Set-Cookie", door_cookie_header(dev["token"]))
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        try:
            self.wfile.write(body)
        except Exception:
            pass

    def do_DELETE(self):
        # The only DELETE on the dashboard surface: removing a
        # paired Mesh peer (spec section 3.1). Behind the same
        # public gate as every other dashboard write.
        if self.path.split("?")[0] == "/api/mesh/peers":
            if self._public_gate_blocks():
                self._send(403, {"error": "access code required"})
                return
            cfg = load_config()
            data = self._body()
            target = str(data.get("nodeId") or "")
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
            borrow["peers"] = [p for p in peers if not (isinstance(p, dict) and str(p.get("nodeId") or "") == target)]
            block["borrow"] = borrow
            cfg["mesh"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        self._send(404, {"ok": False, "error": "not found"})

    def do_GET(self):
        _path = self.path.split("?")[0]
        if _path == "/__gate":
            self._send(200, gate_page_html(approval_mode=bool(door_approved_devices(load_config()))).encode(), "text/html; charset=utf-8")
            return
        if _path == "/__gate/pending":
            # A waiting device's poll — exempt from the gate block
            # below by design: the pending id is its credential.
            self._gate_pending_poll()
            return
        # Build .86: the node-facing Mesh endpoints (/mesh/status,
        # /mesh/borrow) are NOT served on this dashboard surface at
        # all — borrowers dial the dedicated Mesh port (the peer
        # card's port) only. The dashboard-local Mesh routes
        # (/mesh/card, /mesh/lend/stop, /api/mesh/*) stay below,
        # behind the gate, exactly as they were.
        if self._public_gate_blocks():
            if _path.startswith("/api/"):
                self._send(403, {"error": "access code required"})
            else:
                self._send(200, gate_page_html(approval_mode=bool(door_approved_devices(load_config()))).encode(), "text/html; charset=utf-8")
            return
        # Build 9 (#90): GET /mesh/card and /api/mesh/card — the
        # card as copyable text — are REMOVED. Pairing is the PNG
        # picture only; those paths now fall through to this
        # handler's not-found answer like any retired route. The
        # card DATA still exists (mesh_card builds it), but it
        # leaves this machine only inside the PNG below.
        if _path in ("/mesh/card.png", "/api/mesh/card.png"):
            # This node's pairing card as a picture (Build 7,
            # spec section 13; the ONLY card surface since
            # Build 9, #90): a dashboard-surface read behind the
            # public gate above — the DHD proxy never forwards
            # /mesh/* here — answering the PNG whose face is the
            # card and whose payload is the card.
            _cfg = load_config()
            if not mesh_settings(_cfg)["enabled"]:
                self._send(200, {"ok": False, "error": "Apex | Mesh is off on this machine — turn it on from the Mesh page first."})
                return
            _ok, _err = mesh_ensure_identity(_cfg)
            if not _ok:
                self._send(200, {"ok": False, "error": _err})
                return
            _ok, _err = mesh_ensure_cert()
            if not _ok:
                self._send(200, {"ok": False, "error": _err})
                return
            # Build 4 (#143): the generator refuses a card it
            # cannot honestly address — no physical-LAN address
            # and no declared public endpoint — in words, here,
            # instead of minting a picture that strands the pair.
            _refusal = mesh_card_refusal(_cfg)
            if _refusal:
                self._send(200, {"ok": False, "error": _refusal})
                return
            self._send(200, build_mesh_card_png(load_config()), "image/png")
            return
        if _path in ("/favicon-32.png", "/apple-touch-icon.png", "/icon-180.png"):
            # Build 13 (issue #151): the DHD door wears the Row's
            # icon. favicon-scale derivatives of the Nocturne Row
            # site's logo.png, bundled as plain app assets and
            # served behind the same public gate as every other
            # dashboard byte (the #116 portrait precedent below).
            # /apple-touch-icon.png answers the name browsers ask
            # for by default with the 180x180 file; a missing file
            # answers 404, never a crash.
            _icon_name = "favicon-32.png" if _path == "/favicon-32.png" else "icon-180.png"
            try:
                with open(os.path.join(ROOT, "assets", _icon_name), "rb") as _if:
                    _idata = _if.read()
                self._send(200, _idata, "image/png")
            except OSError:
                self._send(404, {"ok": False, "error": "icon not found"})
            return
        if _path == "/sw.js":
            # v0.5.14e (#272): the service worker — an embedded
            # string, not a tree file, so the package's shape
            # and the install contract do not change to carry
            # it. Behind the same public gate as every other
            # dashboard byte; it holds no state of its own.
            self._send(200, PUSH_SERVICE_WORKER_JS.encode(),
                       "text/javascript; charset=utf-8")
            return
        if _path == "/assets/navi-portrait.jpg":
            # Issue #116: Navi's bundled portrait (the lamp-post key
            # art) for the chat presence header and message gutter.
            # A plain app asset behind the same public gate as every
            # other dashboard byte; a missing file answers 404, never
            # a crash.
            try:
                with open(os.path.join(ROOT, "assets", "navi-portrait.jpg"), "rb") as _pf:
                    _pdata = _pf.read()
                self._send(200, _pdata, "image/jpeg")
            except OSError:
                self._send(404, {"ok": False, "error": "portrait not found"})
            return
        if self.path.split("?")[0] in ("/", "/index.html"):
            with open(os.path.join(ROOT, "app", "index.html"), "rb") as f:
                html = f.read()
            # Stamp the served page with the build and the install
            # folder it came from, as an HTML comment at the very top.
            # When the page at localhost:8675 is in doubt, viewing its
            # source settles which program served it: this line means
            # the Navi bridge did; its absence means something else
            # holds the port — and the folder names which install.
            banner = (
                "<!-- Apex dashboard — build " + BRIDGE_BUILD
                + " — served by the bridge in " + ROOT + " -->\n"
            ).encode()
            self._send(200, banner + html, "text/html; charset=utf-8")
        elif self.path.startswith("/api/pathcomplete"):
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            prefix = (qs.get("prefix") or [""])[0]
            self._send(200, {"entries": path_complete(prefix)})
        elif self.path == "/api/status":
            cfg = load_config()
            endpoint = cfg.get("live", {}).get("llamafile_endpoint", "")
            alive = llamafile_alive(endpoint)
            self._send(200, {
                "app": "Navi: Apex",
                "version": app_version(cfg),
                "versionStatus": app_version_status(cfg),
                "bridge": BRIDGE_BUILD,
                # Build 4: the running process's own Edge build
                # sequence, beside its stamp — the apply helper's
                # confirmation reads it after a relaunch, because
                # the version number alone cannot tell one build
                # of a version from the next.
                "buildSeq": SHIPPED_BUILD_SEQ,
                "port": BRIDGE_PORT,
                "dir": ROOT,
                "root": ROOT,
                "update": update_bg_payload(),
                # v0.4.7e (#242): the bell rides the status poll —
                # the throttled payload, so the producers re-run
                # at most once per NOTIF_RECONCILE_SECONDS.
                "notifications": notifications_payload(cfg),
                # v0.4.10e (#254): the sync bar rides the same
                # poll — the bridge's progress and receipt, read
                # from the sync record.
                "sync": sync_state_payload(),
            "presenceExtra": presence_extra_lines(cfg),
                "updaterChannel": updater_channel(cfg),
                "updaterChannelExplicit": updater_channel_explicit(cfg),
                "model": "navi",
                "llamafile_endpoint": endpoint,
                "llamafile_reachable": alive,
                "publicServer": public_state_payload(cfg, self._client_is_loopback()),
                # Build 15 (#161): the running case, read true —
                # the status column shows which case is open the
                # way it shows every other live state.
                "openCase": open_case_payload(cfg),
                "dhd": dhd_state_payload(cfg, self._client_is_loopback()),
                "mesh": mesh_status_payload(cfg, (self.headers.get("X-Apex-Via") or "") == "dhd"),
                # Build 2 (#204): the status column's FediNet
                # sections — one per membership, state from the
                # federation machinery itself.
                "federation": federation_status_slice(cfg),
                # Build 3 (#209): sudo mode's state, read true —
                # the status column and the Settings card both
                # render from this, so the switch, the stored
                # state and the reload view cannot disagree.
                "sudo": sudo_state(cfg),
                "boot": boot_mode_payload(),
            })
        elif self.path.startswith("/api/operations/"):
            _op_id = urllib.parse.unquote(self.path.rsplit("/", 1)[-1])
            _op = op_snapshot(_op_id)
            if _op:
                self._send(200, {"ok": True, "operation": _op})
            else:
                self._send(404, {"ok": False, "error": "operation not found"})
        elif self.path == "/api/hardware":
            self._send(200, {"profile": hardware_profile_view(load_config())})
        elif self.path == "/api/commands":
            self._send(200, {"commands": installed_commands()})
        elif self.path == "/api/persona":
            self._send(200, {
                "soul": read_file(SOUL_PATH),
                "memory": read_file(MEMORY_PATH),
            })
        elif self.path == "/api/guardian/state":
            # Guardian (Build 14, #156): the Utilities face's one
            # read — pressure, headroom, her llamafile's RSS, the
            # top processes, learned offenders, settings, and the
            # recent action log, straight from the engine.
            try:
                self._send(200, GUARDIAN.state())
            except Exception as e:
                self._send(200, {"enabled": False, "pressure": "ok", "freeMb": 0,
                                 "reserveMb": 1024, "llamafileRssMb": 0, "generating": False,
                                 "mesh": {"enabled": False, "peers": 0}, "top": [], "offenders": [],
                                 "expendable": [], "settings": {}, "log": [],
                                 "error": "Guardian state failed: " + str(e)})
        elif self.path == "/api/voice":
            # The Voice tab's state: her settings, whether each
            # speech engine is actually installed (never assumed),
            # and the live call, if one is running. This endpoint
            # must never die silently: a handler exception used to
            # drop the connection, and the dashboard could only say
            # "The bridge didn't answer" — the real reason (a voice
            # venv this Python couldn't read, a broken state file)
            # never reached the operator. Any failure is now
            # reported in the payload itself, as the detail.
            try:
                self._send(200, voice_status(load_config()))
            except Exception as e:
                try:
                    vc = voice_config(load_config())
                except Exception:
                    vc = dict(VOICE_DEFAULTS)
                detail = (
                    "The bridge's voice check itself failed: "
                    + type(e).__name__ + ": " + str(e)
                )
                self._send(200, {
                    "enabled": vc["enabled"],
                    "config": vc,
                    "python": sys.executable,
                    "venv": VOICE_VENV_SITE_PACKAGES,
                    "stt": {"engine": "faster-whisper", "model": vc["stt_model"],
                            "available": False, "detail": detail},
                    "tts": {"engine": vc["tts_engine"], "voice": vc["tts_voice"],
                            "available": False, "detail": detail},
                    "voices": [{"id": vc["tts_voice"], "label": vc["tts_voice"],
                                "source": "current"}],
                    "voices_source": "current",
                    "call": None,
                })
        elif self.path == "/api/voice/calls":
            self._send(200, {"calls": load_voice_calls()})
        elif self.path.split("?")[0] == "/api/chat/route":
            # v0.4.1e (#228): the route pill reads its face from
            # here — the stored route, the effective default, and
            # the access table, in one answer.
            _q = urllib.parse.parse_qs(
                urllib.parse.urlparse(self.path).query)
            _cid = (_q.get("chatId") or [""])[0]
            self._send(200, chat_route_answer(load_config(), _cid))
        elif self.path == "/api/operator/badge-info":
            # Issue #116: the fact an Operator Badge may carry about
            # this install beyond the operator's own profile: the
            # node's display name, public-by-design and self-asserted
            # when it rides the badge. Deliberately NOT the Mesh
            # identity: no node id, no key material, no certificate
            # fingerprint - a badge is a self-portrait, never a
            # credential, and nothing anywhere reads this (or a badge)
            # to authenticate anyone.
            _ms = mesh_settings(load_config())
            self._send(200, {
                "nodeName": str(_ms.get("nodeName") or ""),
            })
        elif self.path == "/api/chips/catalog":
            # Navi User Repo tab: live GitHub search for the navi-chips
            # topic, merged with the installs recorded on this
            # machine. A failed search reports its error and an empty
            # list rather than inventing a catalog.
            try:
                self._send(200, chip_catalog_view())
            except Exception as e:
                self._send(200, {
                    "error": "GitHub search failed: " + str(e),
                    "fetchedAt": None,
                    "chips": [],
                })
        elif self.path.split("?")[0] == "/api/chips/readme":
            # The back of a NUR flip card: the chip repo's README,
            # fetched raw from GitHub on demand. A repo without one
            # (or an unreachable GitHub) reports the reason instead
            # of inventing content.
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            repo = (qs.get("repo") or [""])[0]
            if not repo or "/" not in repo:
                self._send(200, {"ok": False, "error": "No repo was named."})
            else:
                try:
                    md = github_text(
                        "https://api.github.com/repos/" + repo + "/readme",
                        "application/vnd.github.raw+json",
                    )
                    self._send(200, {"ok": True, "markdown": md})
                except urllib.error.HTTPError as e:
                    if e.code == 404:
                        self._send(200, {"ok": False, "error": "This repo has no README."})
                    else:
                        self._send(200, {"ok": False, "error": "GitHub answered HTTP " + str(e.code) + " — the README could not be loaded."})
                except Exception:
                    self._send(200, {"ok": False, "error": "GitHub could not be reached — the README could not be loaded."})
        elif self.path.split("?")[0] == "/api/rag/readme":
            # The back of a pack flip card: the pack repo's README,
            # fetched raw from GitHub on demand — the pack
            # counterpart of /api/chips/readme, same contract. (The
            # POST /api/rag/readme further down is a different
            # thing: the local pack README editor's read.)
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            repo = (qs.get("repo") or [""])[0]
            if not repo or "/" not in repo:
                self._send(200, {"ok": False, "error": "No repo was named."})
            else:
                try:
                    md = github_text(
                        "https://api.github.com/repos/" + repo + "/readme",
                        "application/vnd.github.raw+json",
                    )
                    self._send(200, {"ok": True, "markdown": md})
                except urllib.error.HTTPError as e:
                    if e.code == 404:
                        self._send(200, {"ok": False, "error": "This repo has no README."})
                    else:
                        self._send(200, {"ok": False, "error": "GitHub answered HTTP " + str(e.code) + " — the README could not be loaded."})
                except Exception:
                    self._send(200, {"ok": False, "error": "GitHub could not be reached — the README could not be loaded."})
        elif self.path == "/api/rag/catalog":
            # The RAG tab's Packs list: live GitHub search for the
            # navi-rag topic, merged with the packs installed on
            # this machine and their active flags.
            try:
                self._send(200, rag_catalog_view())
            except Exception as e:
                self._send(200, {
                    "error": "GitHub search failed: " + str(e),
                    "fetchedAt": None,
                    "packs": [],
                })
        elif self.path == "/api/vault/list":
            # The Vault (build .72, issue #54): every note with its
            # exact text, plus folders, wikilinks, tags, backlinks.
            self._send(200, vault_list_payload())
        elif self.path.split("?")[0] == "/api/vault/raw":
            # v0.5.3e (#265): a Vault file's raw bytes — the door
            # the reading views' images come through. Jailed to
            # the Vault by vault_join like every Vault path;
            # raster images answer as images, everything else as
            # plain bytes under a sandbox policy, so a filed
            # file can be SHOWN but never run. Behind the public
            # gate with every other /api/ read above.
            _qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            _rel = vault_join((_qs.get("path") or [""])[0])
            _data, _ctype = (None, None)
            if _rel is not None:
                _data, _ctype = vault_raw_payload(_rel)
            if _data is None:
                self._send(404, {"ok": False, "error": "no such vault file"})
            else:
                self.send_response(200)
                self.send_header("Content-Type", _ctype)
                self.send_header("Content-Length", str(len(_data)))
                self.send_header("X-Content-Type-Options", "nosniff")
                self.send_header("Content-Security-Policy", "sandbox")
                self.end_headers()
                try:
                    self.wfile.write(_data)
                except Exception:
                    pass
        elif self.path.split("?")[0] == "/api/fedichat/media":
            # v0.5.7e (FediChat Build 2, room life): a room
            # image's bytes — the fedichat twin of the vault
            # door above. Jailed twice over: the name must be
            # a blob hash, and the hash must be referenced by
            # a line HELD in the named room on this node (the
            # blob store is never browsable), and the bytes
            # must recompute to the hash before they are
            # served. Behind the public gate with every other
            # /api/ read above.
            _qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            _fed_id = (_qs.get("fed") or [""])[0]
            _room_id = (_qs.get("room") or [""])[0]
            _sha = (_qs.get("sha") or [""])[0]
            _data, _ctype = (None, None)
            if re.fullmatch(r"[0-9a-f]{64}", _sha or ""):
                _cfgm = load_config()
                _im, _fedm = _fed_find(federation_settings(_cfgm), _fed_id)
                if _fedm is not None:
                    for _lm in fedichat_log_read(_fed_id, _room_id):
                        _media = _lm.get("media")
                        if isinstance(_media, dict) and str(_media.get("sha") or "") == _sha:
                            _data = fedichat_media_read(_sha)
                            _ctype = str(_media.get("type") or "application/octet-stream")
                            break
            if _data is None:
                self._send(404, {"ok": False, "error": "no such room image"})
            else:
                self.send_response(200)
                self.send_header("Content-Type", _ctype)
                self.send_header("Content-Length", str(len(_data)))
                self.send_header("X-Content-Type-Options", "nosniff")
                self.send_header("Content-Security-Policy", "sandbox")
                self.end_headers()
                try:
                    self.wfile.write(_data)
                except Exception:
                    pass
        elif self.path.split("?")[0] == "/api/vault/search":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            self._send(200, {"hits": vault_search((qs.get("q") or [""])[0], 8)})
        elif self.path == "/api/vault/export":
            # The whole folder as a .zip — unzip it and Obsidian
            # opens the result as a vault, exactly as it lies here.
            vault_ensure_seed()
            buf = io.BytesIO()
            with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as z:
                if os.path.isdir(VAULT_DIR):
                    for base, dirs, files in os.walk(VAULT_DIR):
                        dirs.sort()
                        for name in sorted(files):
                            full = os.path.join(base, name)
                            z.write(full, os.path.relpath(full, VAULT_DIR).replace(os.sep, "/"))
            self._send(200, buf.getvalue(), "application/zip")
        elif self.path == "/api/chats":
            # Server-side chats (#139): the whole store — threads
            # and the uploads index — for the window to hydrate.
            self._send(200, chats_payload())
        elif self.path == "/api/intel":
            # Threat Intelligence Center (Build 15, #165): the
            # Apps surface's whole picture in one payload —
            # freshness first, the triage queue, the brief.
            try:
                self._send(200, INTEL.state())
            except Exception as e:
                self._send(200, {"ok": False, "error": "The intel store could not be read: " + str(e)[:160], "sources": [], "queue": [], "advisories": [], "watchlist": []})
        elif self.path.split("?")[0] == "/api/intel/item":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            _item = None
            try:
                _item = INTEL.item((qs.get("cve") or [""])[0])
            except Exception:
                _item = None
            if _item is None:
                self._send(404, {"ok": False, "error": "Not in the local intel store."})
            else:
                self._send(200, {"ok": True, "item": _item})
        elif self.path == "/api/wire":
            self._send(200, WIRE.state())
        elif self.path == "/api/pulse":
            # v0.5.13e (#287): the Pulses tab's read — watches
            # with their coverage, the enabled networks as they
            # stand, and the cases a watch can feed.
            self._send(200, pulse_payload(load_config()))
        elif self.path == "/api/day":
            # v0.6.0e (#294/#295/#296): the Day surface's read —
            # the book as it stands.
            self._send(200, daybook_payload(load_config()))
        elif self.path == "/api/greeting":
            # v0.5.13e (#288): an explicit composition — the
            # rotation advances, a fresh pulse check folds in,
            # and the digest it speaks is marked greeted.
            self._send(200, greeting_payload(load_config()))
        elif self.path.split("?")[0] == "/api/seraph/state":
            self._send(200, seraph_payload())
        elif self.path.split("?")[0] == "/api/shell/session/state":
            # CLI Mode (Build 3): the session's events since the
            # page's last sequence number, plus the full-screen
            # grid while a program holds the alternate screen.
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            try:
                _since = int((qs.get("since") or ["0"])[0])
            except Exception:
                _since = 0
            self._send(200, shell_session_state(
                (qs.get("chatId") or ["main"])[0], _since))
        elif self.path == "/api/sudo/audit":
            # The record, read-only: every elevated command Navi
            # has run under sudo mode. It lives in the install
            # home and never rides Sync.
            self._send(200, {"ok": True, "entries": sudo_audit_entries(100),
                             "sudo": sudo_state()})
        elif self.path.split("?")[0] == "/api/wire/article":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            self._send(200, WIRE.read_article((qs.get("url") or [""])[0]))
        elif self.path == "/api/ctf":
            self._send(200, CTF.state())
        elif self.path == "/api/bounties":
            self._send(200, BOUNTIES.state())
        elif self.path == "/api/intel/digest":
            try:
                self._send(200, INTEL.build_digest())
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e)[:160]})
        elif self.path == "/api/notifications":
            # v0.4.7e (#242): the bell's own read — a forced
            # reconcile, so opening the panel always answers from
            # the world as it stands now.
            self._send(200, notifications_payload(load_config(), force=True))
        elif self.path == "/api/push/state":
            # v0.5.14e (#272): the push doors read this — the
            # node's VAPID public key (minted once, below) and
            # the phones subscribed to THIS node, labels only:
            # endpoints and key material never ride a read.
            self._send(200, push_state_payload(load_config()))
        elif self.path == "/api/cases":
            self._send(200, cases_list_payload(
                (self.headers.get("X-Apex-Via") or "") == "dhd"))
        elif self.path.split("?")[0] == "/api/cases/detail":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            _detail = case_detail_payload((qs.get("case") or [""])[0])
            if _detail is None:
                self._send(404, {"ok": False, "error": "No such case."})
            else:
                self._send(200, {"ok": True, "case": _detail})
        elif self.path.split("?")[0] == "/api/rag/cartridge.png":
            # The pack as a picture, built on demand from the pack
            # folder (issue #93): the sleeve image the RAG tab and
            # ChipDeck show, and the file a download saves.
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            slug = os.path.basename((qs.get("slug") or [""])[0])
            payload = rag_cartridge_payload(slug) if slug else None
            if payload is None:
                self._send(404, {"error": "not found"})
            else:
                png = build_rag_cartridge_png(payload)
                if len(png) > RAG_CARTRIDGE_MAX_BYTES:
                    self._send(404, {"error": "too big for a picture"})
                else:
                    self._send(200, png, "image/png")
        elif self.path == "/api/rag/packs":
            self._send(200, {"packs": local_rag_packs()})
        elif self.path.split("?")[0] == "/api/rag/pack-source":
            # RAG Studio opens a pack from the folder for editing:
            # its name, description, and every document with its
            # body — forged, installed, or an exported bundle that
            # came back as an install alike.
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            slug = os.path.basename((qs.get("slug") or [""])[0])
            pack = rag_pack_documents(slug) if slug else None
            if pack:
                self._send(200, {"ok": True, "slug": slug, "name": pack["name"], "description": pack["description"], "documents": pack["documents"], "icon": valid_icon_art((_rag_pack_index(slug) or {}).get("icon")) or []})
            else:
                self._send(200, {"ok": False, "error": "That pack is not in the folder."})
        elif self.path.split("?")[0] == "/api/rag/export-file":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            name = os.path.basename((qs.get("file") or qs.get("name") or [""])[0])
            path = os.path.join(RAG_EXPORTS_DIR, name)
            if name.endswith(".png") and os.path.isfile(path):
                with open(path, "rb") as f:
                    self._send(200, f.read(), "image/png")
            elif name.endswith(".zip") and os.path.isfile(path):
                with open(path, "rb") as f:
                    self._send(200, f.read(), "application/zip")
            else:
                self._send(404, {"error": "not found"})
        elif self.path.split("?")[0] == "/api/chips/export-file":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            name = os.path.basename((qs.get("file") or qs.get("name") or [""])[0])
            path = os.path.join(CHIPS_EXPORTS_DIR, name)
            if name.endswith(".zip") and os.path.isfile(path):
                with open(path, "rb") as f:
                    self._send(200, f.read(), "application/zip")
            else:
                self._send(404, {"error": "not found"})
        elif self.path.split("?")[0] == "/api/chips/file":
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            name = os.path.basename((qs.get("file") or qs.get("name") or [""])[0])
            path = os.path.join(CHIPS_DIR, name)
            if name.endswith(".png") and os.path.isfile(path):
                with open(path, "rb") as f:
                    self._send(200, f.read(), "image/png")
            else:
                self._send(404, {"error": "not found"})
        elif self.path == "/api/deck":
            # ChipDeck (0.1.9e, #94): the binder's contents — every
            # picture-riding thing on this machine, with its state.
            # (Loads its own config: this route sits above the
            # handler's shared cfg assignment.)
            self._send(200, deck_contents(load_config()))
        elif self.path.split("?")[0] == "/api/deck/file":
            # A Mesh pairing card's picture, for its ChipDeck
            # sleeve (chips ride /api/chips/file, packs ride
            # /api/rag/cartridge.png). Basename-jailed to the
            # cards folder, .png only.
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            name = os.path.basename((qs.get("name") or [""])[0])
            path = os.path.join(MESH_CARDS_DIR, name)
            if name.endswith(".png") and os.path.isfile(path):
                with open(path, "rb") as f:
                    self._send(200, f.read(), "image/png")
            else:
                self._send(404, {"error": "not found"})
        elif self.path == "/api/workflows":
            # Workflows tab: the saved chains plus every chip in
            # the folder a step can point at.
            self._send(200, {
                "workflows": load_workflows(),
                "chips": [{"file": c["file"], "name": c["name"], "source": c.get("source"), "sourceKey": c.get("sourceKey"), "locallyEdited": bool(c.get("locallyEdited")), "unreadable": bool(c.get("unreadable")), "requirements": list(c.get("requirements") or [])} for c in local_chips()],
            })
        elif self.path.split("?")[0] == "/api/chips/source":
            # Chip Studio opens a local chip for editing: its
            # full payload (name, description, code, output).
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            name = os.path.basename((qs.get("file") or qs.get("name") or [""])[0])
            path = os.path.join(CHIPS_DIR, name)
            payload = None
            if name.endswith(".png") and os.path.isfile(path):
                try:
                    with open(path, "rb") as f:
                        payload = parse_cartridge_png(f.read())
                except Exception:
                    payload = None
            if payload:
                payload["ok"] = True
                payload["file"] = name
                self._send(200, payload)
            else:
                self._send(200, {"ok": False, "error": "That chip could not be read."})
        elif self.path == "/api/github":
            # The GitHub account state in one answer: URLs, client-id
            # configuration, sign-in, backup settings, and the last
            # backup/import records. (Settings → GitHub & Backup reads
            # it; the GitHub section itself is the project pulse.)
            self._send(200, github_status_payload(load_config()))
        elif self.path == "/api/sync":
            # Build 3 (#176): the Sync card's resting state — when
            # the last round ran and what it said, so reopening the
            # card (or the app) shows the truth without a round.
            self._send(200, sync_state_payload())
        elif self.path == "/api/github/issues":
            # The Issues tab: live open issues for the project repo.
            self._send(200, github_issues_payload())
        elif self.path == "/api/github/pulse":
            # The Overview tab: the project's live pulse.
            self._send(200, github_pulse_payload())
        elif self.path.split("?")[0] == "/api/update/check":
            # The Updater's real check: the newest GitHub release
            # on the install's channel against the VERSION file. A
            # check that cannot complete answers ok: False with
            # the reason — never a made-up "up to date".
            qs = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            chan = (qs.get("channel") or [""])[0]
            self._send(200, update_check_payload(
                load_config(), chan if chan in ("stable", "edge") else None))
        elif self.path == "/api/update/result":
            # The end-to-end apply's live record: its current step
            # while it runs (written by the bridge's worker, then by
            # the detached helper once the install takes the bridge
            # down), and afterwards the confirmed version before →
            # after — or the failed step, its reason, and the manual
            # fallback. The relaunched bridge serves the same file,
            # which is how the panel learns the outcome.
            self._send(200, update_result_payload())
        else:
            self._send(404, {"error": "not found"})

    def do_POST(self):
        cfg = load_config()
        if self.path.split("?")[0] == "/__gate":
            # The gate page's form: a remote visitor proving the
            # code. Since Build 2 (#173) the code knocks rather
            # than opens: the registry decides — a session with the
            # device's OWN 30-day token (the bootstrap first device,
            # or a trusted device renewing), or a PENDING device
            # and its waiting room, approved from a trusted session
            # in Settings -> System -> Door. Failure serves the
            # gate page again with the reason, and nothing else.
            # v0.6.7e (#313, leg 1): this door is throttled
            # like the DHD door — the same dhd_throttle_*
            # helpers and constants, keyed on the socket peer.
            # (Legs 2 and 3 are NOT in this build: the
            # first-device bootstrap is #173 doctrine awaiting
            # Alex's ruling, and proxy-aware client IPs are a
            # design item of their own.)
            _gate_ip = ""
            try:
                _gate_ip = str(self.client_address[0] or "")
            except Exception:
                _gate_ip = ""
            _gfails, _glock = dhd_throttle_state(_gate_ip)
            if _glock > time.time():
                self._send(
                    429,
                    gate_page_html(
                        "Too many wrong codes - this door is locked.",
                        lock_seconds=int(_glock - time.time()),
                        approval_mode=bool(door_approved_devices(cfg)),
                    ).encode(),
                    "text/html; charset=utf-8",
                )
                return
            length = int(self.headers.get("Content-Length") or 0)
            raw = self.rfile.read(length).decode("utf-8", "replace") if length else ""
            form = urllib.parse.parse_qs(raw)
            given = (form.get("code") or [""])[0].strip().upper()
            dev_name = (form.get("device_name") or [""])[0].strip()
            want = public_settings(cfg)["access_code"]
            if want and given == want.upper():
                dhd_throttle_clear(_gate_ip)
                action, dev = door_gate_outcome(cfg, dev_name, self.headers.get("User-Agent"), door_cookie_value(self.headers.get("Cookie")))
                if action == "session":
                    self.send_response(303)
                    self.send_header("Location", "/")
                    self.send_header("Set-Cookie", door_cookie_header(dev["token"]))
                    self.send_header("Content-Length", "0")
                    self.end_headers()
                else:
                    self._send(200, gate_pending_html(dev["id"], dev["name"], door="node").encode(), "text/html; charset=utf-8")
            else:
                _gfails = dhd_throttle_fail(_gate_ip)
                if _gfails >= DHD_LOCKOUT_FAILS:
                    self._send(
                        429,
                        gate_page_html(
                            "Too many wrong codes - this door is now locked.",
                            lock_seconds=DHD_LOCKOUT_SECONDS,
                            approval_mode=bool(door_approved_devices(cfg)),
                        ).encode(),
                        "text/html; charset=utf-8",
                    )
                    return
                time.sleep(min(2 ** (_gfails - 1), 8))
                self._send(
                    200,
                    gate_page_html(
                        "That code did not open the door - check the code on the DHD page of the machine Apex runs on, and try again.",
                        attempts_remaining=max(0, DHD_LOCKOUT_FAILS - _gfails),
                        approval_mode=bool(door_approved_devices(cfg)),
                    ).encode(),
                    "text/html; charset=utf-8",
                )
            return
        # Build .86: /mesh/borrow is not served on this surface
        # (see do_GET) — the dedicated Mesh port is the only
        # lender door.
        if self._public_gate_blocks():
            self._send(403, {"error": "access code required"})
            return
        if self.path.split("?")[0] == "/mesh/lend/stop":
            # Instant-off (R4): a LOCAL dashboard action only. It
            # is never served on the DHD-forwarded surface — a
            # request the proxy carried in is refused here even
            # with a valid dashboard session, and no node signature
            # can reach it either (this route is behind the gate,
            # not the signature check).
            if (self.headers.get("X-Apex-Via") or "") == "dhd":
                self._send(403, {"ok": False, "error": "Stop lending now is a local action — it is not available through the DHD door."})
                return
            self._send(200, {"ok": True, "mesh": mesh_stop_lending((self.headers.get("X-Apex-Via") or "") == "dhd")})
            return
        if self.path == "/api/guardian/settings":
            # Guardian (Build 14, #156): the Utilities face's save.
            # Whitelisted keys only, merged into the config's
            # guardian block (the dhd_save / mesh-settings idiom);
            # the engine reads settings fresh every tick, so the
            # save is live on the next sample. The answer is the
            # new state, so the face renders what is actually true.
            data = self._body()
            block = cfg.get("guardian") if isinstance(cfg.get("guardian"), dict) else {}
            block = dict(block)
            if isinstance(data.get("enabled"), bool):
                block["enabled"] = data["enabled"]
            if isinstance(data.get("killEnabled"), bool):
                block["killEnabled"] = data["killEnabled"]
            for _gk in ("reserveMb", "slowGenSeconds", "cpuCapPercent", "warnAfterSeconds", "killGraceSeconds"):
                try:
                    block[_gk] = int(data[_gk])
                except Exception:
                    pass
            try:
                block["sampleSeconds"] = float(data["sampleSeconds"])
            except Exception:
                pass
            if isinstance(data.get("expendable"), list):
                block["expendable"] = [str(x).strip() for x in data["expendable"] if str(x).strip()]
            cfg["guardian"] = block
            save_config(cfg)
            if block.get("enabled") is not False:
                try:
                    GUARDIAN.start()
                except Exception:
                    pass
            self._send(200, {"ok": True, "state": GUARDIAN.state()})
            return
        if self.path in ("/api/boot/arm", "/api/boot/disarm"):
            # Arm at boot (Build 14, #159): a LOCAL dashboard
            # action only — it rewrites this machine's boot
            # behavior, so it is never served through the DHD
            # door, the same refusal as Stop-lending above.
            if (self.headers.get("X-Apex-Via") or "") == "dhd":
                self._send(403, {"ok": False, "error": "Arming at boot is a local action — it is not available through the DHD door."})
                return
            self._send(200, boot_set_armed(self.path == "/api/boot/arm"))
            return
        if self.path == "/api/mesh/settings":
            # The Settings → System Mesh card's save. Persists the
            # block, sets identity up on first enable, and starts
            # or stops the dedicated Mesh door to match. Q1: lending
            # may be turned ON only in person — a save arriving
            # through the DHD proxy surface is refused for that one
            # change; turning anything OFF is allowed from anywhere
            # the operator is signed in.
            data = self._body()
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            lend = block.get("lend") if isinstance(block.get("lend"), dict) else {}
            lend = dict(lend)
            was_enabled = block.get("enabled") is True
            was_lending = lend.get("enabled") is True
            want_lending = bool(data.get("lendEnabled")) if "lendEnabled" in data else was_lending
            if want_lending and not was_lending and (self.headers.get("X-Apex-Via") or "") == "dhd":
                self._send(200, {"ok": False, "error": "Lending can only be turned on from this machine's own dashboard, in person — not through the DHD door. Turning it off works from anywhere."})
                return
            if "enabled" in data:
                block["enabled"] = bool(data.get("enabled"))
            if "nodeName" in data:
                block["nodeName"] = str(data.get("nodeName") or "")[:80]
            # Build 4 (#154): the operator-declared public
            # endpoint. Typed, never harvested; canonicalized to
            # host:port here, and a value that is not a host:port
            # refuses the whole save in words rather than landing
            # as junk on the card.
            if "publicEndpoint" in data:
                _ep_err = _mesh_endpoint_error(data.get("publicEndpoint"))
                if _ep_err:
                    self._send(200, {"ok": False, "error": _ep_err})
                    return
                block["publicEndpoint"] = _mesh_endpoint_clean(data.get("publicEndpoint"))
            if isinstance(data.get("listenPorts"), list):
                block["listenPorts"] = _mesh_listen_ports(
                    data["listenPorts"], block.get("port", 8765))
            if "borrowEnabled" in data:
                borrow["enabled"] = bool(data.get("borrowEnabled"))
            if "graceMs" in data:
                borrow["graceMs"] = max(0, min(30000, _mesh_int(data.get("graceMs"), 2000)))
            if "lendEnabled" in data:
                lend["enabled"] = want_lending
            # Build 7 (#87, Fix F): the lending caps are operator
            # settings on the lending machine — share of each hour
            # (1–100%), borrowed generations an hour and per peer
            # an hour (1–10000 each). They persist in the lend
            # block and take effect on the next admission decision
            # and the next card/capability report, because both
            # read mesh_settings(load_config()) fresh — no bridge
            # restart. The shipped defaults (25 / 20 / 10) are
            # only the fallbacks for a missing or unparseable
            # value, here and in mesh_settings.
            if "maxSharePercent" in data:
                lend["maxSharePercent"] = max(1, min(100, _mesh_int(data.get("maxSharePercent"), 25)))
            if "maxGenerationsPerHour" in data:
                lend["maxGenerationsPerHour"] = max(1, min(10000, _mesh_int(data.get("maxGenerationsPerHour"), 20)))
            if "perPeerPerHour" in data:
                lend["perPeerPerHour"] = max(1, min(10000, _mesh_int(data.get("perPeerPerHour"), 10)))
            # Build 7 (Fix F2): the idle-only override rides the
            # same save. Admission already reads lend["idleOnly"]
            # on every request, so the flip is live at once; the
            # shipped default stays ON (a lender serves only
            # after idleAfterSeconds of quiet) until the operator
            # turns it off here.
            if "idleOnly" in data:
                lend["idleOnly"] = bool(data.get("idleOnly"))
            if "ledgerInBackup" in data:
                block["ledgerInBackup"] = bool(data.get("ledgerInBackup"))
            block["borrow"] = borrow
            block["lend"] = lend
            cfg["mesh"] = block
            if block.get("enabled") is True and not was_enabled:
                # Build 7 (#87): a node whose name was never set
                # sits "Unnamed" in every roster — at enable time,
                # a blank name takes the machine's hostname as its
                # default, persisted with this save. A name the
                # operator typed is never overwritten.
                if not str(block.get("nodeName") or "").strip():
                    block["nodeName"] = socket.gethostname()[:80]
                _ok, _err = mesh_ensure_identity(cfg)
                if not _ok:
                    self._send(200, {"ok": False, "error": _err})
                    return
                _ok, _err = mesh_ensure_cert()
                if not _ok:
                    self._send(200, {"ok": False, "error": _err})
                    return
                # Pick up the nodeId identity setup recorded — but
                # identity saves the config only when it writes a
                # NEW nodeId, so on a re-enable (identity already on
                # disk) the file still holds the pre-save block, and
                # reloading it wholesale would resurrect that block
                # and silently drop this save's change while still
                # answering ok: the master switch stayed off under
                # a "Saved." (field report, 0.1.8e Build 2 — the
                # first enable worked precisely because identity's
                # first write carried the change with it). The
                # operator's block is the change being saved; the
                # saved config contributes only the nodeId it holds.
                _saved = load_config().get("mesh")
                if isinstance(_saved, dict) and _saved.get("nodeId"):
                    block["nodeId"] = _saved["nodeId"]
                cfg["mesh"] = block
            save_config(cfg)
            if block.get("enabled") is True:
                mesh_start_listener()
                mesh_tunnel_sync()
                if not was_enabled:
                    # Build 7 (#87): enabling Mesh checks every
                    # paired peer at once, not at the health
                    # cadence's leisure.
                    mesh_probe_peers_soon()
            else:
                mesh_stop_listener()
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config(), (self.headers.get("X-Apex-Via") or "") == "dhd")})
            return
        # Build 9 (#90): POST /api/mesh/peers — the paste-a-text-
        # card import — is REMOVED. Pairing is the PNG picture
        # only (the /api/mesh/peers/png route below); a request
        # to the retired path falls through to this handler's
        # not-found answer. Peers paired earlier are stored
        # records in config and are untouched by the removal.
        if self.path == "/api/mesh/health":
            # 0.2.0e Build 1 (#143): the operator-triggered Mesh
            # health check \u2014 the two decisive reads of the
            # field diagnosis, run by the product itself. It
            # changes no pairing, pin, or routing state; it only
            # reports. The probes inside it are the short-timeout
            # kind, so the answer lands in seconds.
            self._send(200, {"ok": True, "health": mesh_health_check(cfg)})
            return
        if self.path == "/api/mesh/peers/png":
            # Add a peer from a PNG picture card (Build 7, spec
            # section 13). The dashboard sends the picture's bytes
            # base64'd — from the file-upload control or a drag-
            # and-drop, both through the one client importer — and
            # the payload inside the picture goes through the same
            # card validation and the same add every pairing
            # passes: one parser, one pairing, the identical
            # peer record.
            # A picture with no readable payload, or a damaged
            # one, is refused with its specific reason — never a
            # silent no-op.
            data = self._body()
            raw = None
            try:
                raw = base64.b64decode(str(data.get("png") or ""), validate=True)
            except Exception:
                raw = None
            if not raw:
                self._send(200, {"ok": False, "error": "That upload did not arrive as a picture — try the upload again."})
                return
            if len(raw) > 8 * 1024 * 1024:
                self._send(200, {"ok": False, "error": "That file is far larger than a Mesh card picture — the cards are small PNGs. Check you picked the card file, and try again."})
                return
            peer, png_err = parse_mesh_card_png(raw)
            if peer is None:
                self._send(200, {"ok": False, "error": png_err})
                return
            aerr = mesh_add_peer(cfg, peer)
            if aerr:
                self._send(200, {"ok": False, "error": aerr})
                return
            # Build 6 (#143 ask 3): the pairing the first-run
            # certificate notice accompanied has landed — the
            # notice has been seen with a pin, and is not shown
            # again on this machine.
            mesh_mark_cert_notice(cfg)
            mesh_probe_peers_soon()
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/mesh/peers/remove":
            data = self._body()
            target = str(data.get("nodeId") or "")
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
            borrow["peers"] = [p for p in peers if not (isinstance(p, dict) and str(p.get("nodeId") or "") == target)]
            block["borrow"] = borrow
            cfg["mesh"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/mesh/peers/repair":
            # Build 6 (#143): the health check's heal, in place —
            # strip the dead-end addresses from one peer's stored
            # record. The pairing, its pin, its tunnel route and
            # its history are kept; the condition is strict (every
            # address a dead end) and the refusal is a sentence.
            data = self._body()
            ok_r, sentence_r = mesh_repair_peer_record(
                cfg, str(data.get("nodeId") or ""))
            if ok_r:
                mesh_probe_peers_soon()
            self._send(200, {"ok": ok_r,
                             "message" if ok_r else "error": sentence_r,
                             "mesh": mesh_status_payload(load_config()),
                             "health": mesh_health_check(load_config())})
            return
        if self.path == "/api/mesh/peers/can-borrow":
            # The borrow-side toggle for one paired peer (Build 7,
            # #87): whether THIS machine borrows from that peer.
            # It no longer gates the peer's inbound requests —
            # pairing is that side's consent (mesh_authenticate).
            data = self._body()
            target = str(data.get("nodeId") or "")
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
            new_peers = []
            for p in peers:
                if isinstance(p, dict) and str(p.get("nodeId") or "") == target:
                    p = dict(p)
                    p["canBorrow"] = bool(data.get("canBorrow"))
                new_peers.append(p)
            borrow["peers"] = new_peers
            block["borrow"] = borrow
            cfg["mesh"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/mesh/peers/relay":
            # 0.2.0e Build 7 (#154): one relay consent tick. The
            # three kinds name whose consent the tick is, and the
            # tick is stored on the record of the peer it names
            # first (see _mesh_peer): carryTo = "this peer may
            # carry MY envelopes to <other>", acceptFrom = "this
            # peer may carry envelopes TO ME from <other>",
            # willCarryTo = "I will carry THIS peer's envelopes
            # to <other>". All three are consent, so the doctrine
            # is the lending doctrine: granting is an in-person
            # act on this machine's own dashboard — a request
            # through the DHD door may UNtick (revoking works
            # from anywhere) but never tick on.
            data = self._body()
            target = str(data.get("nodeId") or "")
            other = str(data.get("other") or "")
            kind = str(data.get("kind") or "")
            field = {"carryTo": "relayCarryTo",
                     "acceptFrom": "relayAcceptFrom",
                     "willCarryTo": "relayWillCarryTo"}.get(kind)
            if field is None or not target or not other or other == target:
                self._send(200, {"ok": False, "error": "That relay consent change does not name a carrier, another machine, and which consent it is."})
                return
            want = bool(data.get("on"))
            if want and (self.headers.get("X-Apex-Via") or "") == "dhd":
                self._send(200, {"ok": False, "error": "Relay consent is given at this machine, in person — this view is through the DHD door, so the tick cannot be set from here. Unticking works from anywhere."})
                return
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
            known = {str(p.get("nodeId") or "") for p in peers if isinstance(p, dict)}
            if target not in known or other not in known:
                self._send(200, {"ok": False, "error": "That machine is not paired here."})
                return
            new_peers = []
            for p in peers:
                if isinstance(p, dict) and str(p.get("nodeId") or "") == target:
                    p = dict(p)
                    cur = [str(v) for v in (p.get(field) or []) if str(v)]
                    if want and other not in cur:
                        cur.append(other)
                    if not want:
                        cur = [v for v in cur if v != other]
                    p[field] = cur
                new_peers.append(p)
            borrow["peers"] = new_peers
            block["borrow"] = borrow
            cfg["mesh"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/mesh/peers/tunnel":
            # Build 4 (#178): one peer's managed backup route —
            # ordinary borrower-side config (it points THIS
            # machine's ssh at the peer's SSH door; it changes
            # no consent on the peer). The supervisor reconciles
            # as soon as the save lands.
            data = self._body()
            target = str(data.get("nodeId") or "")
            block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
            block = dict(block)
            borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
            borrow = dict(borrow)
            peers = borrow.get("peers") if isinstance(borrow.get("peers"), list) else []
            found = None
            new_peers = []
            for p in peers:
                if isinstance(p, dict) and str(p.get("nodeId") or "") == target:
                    p = dict(p)
                    tun = dict(p.get("tunnel") or {})
                    if "enabled" in data:
                        tun["enabled"] = bool(data.get("enabled"))
                    if "sshHost" in data:
                        tun["sshHost"] = str(data.get("sshHost") or "").strip()[:255]
                    if "sshUser" in data:
                        tun["sshUser"] = str(data.get("sshUser") or "").strip()[:64]
                    if "sshPort" in data:
                        tun["sshPort"] = max(1, min(65535, _mesh_int(data.get("sshPort"), 227)))
                    p["tunnel"] = tun
                    found = tun
                new_peers.append(p)
            if found is None:
                self._send(200, {"ok": False, "error": "That machine is not paired here."})
                return
            if found.get("enabled") and (not found.get("sshHost") or not found.get("sshUser")):
                self._send(200, {"ok": False, "error": "The backup route needs that machine's SSH host and user before it can be turned on — fill both in and save again."})
                return
            borrow["peers"] = new_peers
            block["borrow"] = borrow
            cfg["mesh"] = block
            save_config(cfg)
            mesh_tunnel_sync()
            self._send(200, {"ok": True, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/federation":
            # 0.2.0e Build 8 (#154): the Federation page's acts —
            # fedinets, gateway cards, trust toggles and ticks,
            # founding, caps, the Badge enrollment, the audit
            # log's state. The tiering lives in federation_act:
            # read + revoke ride any door; manage acts through
            # DHD need the Badge step-up; crown acts need the
            # full ceremony at every door. DHD manages only THIS
            # node's federation posture — the acts all read and
            # write this node's own config, so no door can flip
            # another operator's switches.
            data = self._body()
            via_dhd = (self.headers.get("X-Apex-Via") or "") == "dhd"
            self._send(200, federation_act(data, via_dhd))
        if self.path == "/api/fedichat":
            # FediChat (v0.5.3e, #267): read + post are the
            # operator's own speech (not a crown act) — the
            # trusted-device session governs via DHD, exactly
            # like the federation reads above.
            data = self._body()
            via_dhd = (self.headers.get("X-Apex-Via") or "") == "dhd"
            self._send(200, fedichat_act(data, via_dhd))
            return
        if self.path == "/api/dossier":
            # The Dossier (v0.5.6e, #260): the operator's own
            # people-finding work on their own node — via DHD
            # it rides the trusted-device session like the
            # FediChat door, no Badge step-up.
            data = self._body()
            via_dhd = (self.headers.get("X-Apex-Via") or "") == "dhd"
            self._send(200, dossier_act(data, via_dhd))
            return
        if self.path == "/api/mesh/peers/tunnel-authorize":
            # Build 4 (#178) made this act in-person-only, the
            # lending doctrine of its day. The crown-acts ruling
            # (Alex, 2026-11-13 20:56 EST) amended the door while
            # keeping its property — a stolen session must not
            # quietly grant trust: authorizing a tunnel key is a
            # CROWN ACT. Through DHD it runs under the full
            # ceremony (the trusted-device session the door
            # itself provides + the Operator Badge step-up + the
            # named-action confirm + an audit entry); at the desk
            # it needs the same named confirm + audit, the Badge
            # being the door's addition. REVOKING stays open
            # through every door — revocation is never gated —
            # and audits too. The act itself is unchanged:
            # authorizing writes a forwarding-only line for the
            # peer's tunnel key into this operator's
            # ~/.ssh/authorized_keys; revoking removes it.
            via_dhd = (self.headers.get("X-Apex-Via") or "") == "dhd"
            data = self._body()
            target = str(data.get("nodeId") or "")
            action = "revoke" if str(data.get("action") or "") == "revoke" else "authorize"
            ms_now = mesh_settings(cfg)
            peer = next((p for p in ms_now["borrow"]["peers"] if p["nodeId"] == target), None)
            if peer is None:
                self._send(200, {"ok": False, "error": "That machine is not paired here."})
                return
            if not (peer.get("tunnelPublicKey") or ""):
                self._send(200, {"ok": False, "error": "That machine's card carries no tunnel key — it was minted before the managed backup route, or by an older build. Re-share its card from its Mesh page and pair it again, and the key arrives with the card."})
                return
            if action == "authorize":
                ok_c, err_c = _fed_crown_ok(cfg, via_dhd, data,
                                            _fed_confirm_tunnel(peer["name"]))
                if not ok_c:
                    self._send(200, {"ok": False, "error": err_c})
                    return
            ok, sentence = mesh_tunnel_authorize(peer["tunnelPublicKey"], target, action, ms_now["port"])
            if ok:
                federation_audit(
                    "tunnel-" + action, "",
                    ("Authorized " if action == "authorize" else "Revoked ")
                    + peer["name"] + "'s tunnel key"
                    + (" under the crown ceremony." if action == "authorize"
                       else " — revocation, open through every door."),
                    "dhd" if via_dhd else "desk")
            self._send(200, {"ok": ok, "message": sentence, "mesh": mesh_status_payload(load_config())})
            return
        if self.path == "/api/settings/updater-channel":
            # The Updater's channel switch (build .83): persist
            # the operator's Stable / Edge choice in the config's
            # updater block. Checks and the apply path both read
            # it from there, so the choice survives restarts and
            # a remote (DHD) session flips the same switch.
            data = self._body()
            chan = data.get("channel")
            if chan not in ("stable", "edge"):
                self._send(200, {"ok": False, "error": "The channel must be 'stable' or 'edge'."})
                return
            block = cfg.get("updater") if isinstance(cfg.get("updater"), dict) else {}
            block = dict(block)
            block["channel"] = chan
            # Build 2: this save is the operator's own act, so it
            # stamps the choice as explicit — the stamp is what
            # separates a chosen channel from one the config
            # merely inherited (see updater_channel_explicit).
            block["channelExplicit"] = True
            cfg["updater"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "channel": chan, "channelExplicit": True})
            return
        if self.path == "/api/settings/sync-choice":
            # The sync override's setting (v0.6.2e, #303):
            # persist the operator's choice in the config's sync
            # block. The answer carries the value that now
            # stands, so the Settings switch renders the stored
            # truth — and a failed save answers with its refusal,
            # the switch unmoved.
            data = self._body()
            want = data.get("choiceOnPress")
            if not isinstance(want, bool):
                self._send(200, {"ok": False, "error": "The choice must be true or false."})
                return
            stood = sync_set_choice(want, load_config())
            if stood is None:
                self._send(200, {"ok": False, "error": "The choice could not be saved — the config did not take the write, so the setting stands as it was."})
                return
            self._send(200, {"ok": True, "choiceOnPress": stood})
            return
        if self.path == "/api/settings/public":
            # The Local server toggle itself (build .61 — one
            # action): persist the choice in the config block AND
            # restart this bridge so the choice takes effect. The
            # restart rides the exit-75 handshake: the request is
            # recorded in the block, the response goes out, and this
            # process exits 75 — the launcher's supervisor relaunches
            # it, and the new bridge binds per the saved choice and
            # verifies its own bind against it. The running bridge
            # never hot-swaps its bind. Flipping to the state already
            # live needs no restart and confirms on the spot.
            data = self._body()
            block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
            block = dict(block)
            block["enabled"] = bool(data.get("enabled"))
            cfg["public_server"] = block
            ensure_access_code(cfg)
            save_config(cfg)
            want_public = bool(data.get("enabled"))
            live_public = ACTIVE_BIND_HOST == "0.0.0.0"
            if want_public == live_public:
                set_public_restart(cfg, "confirmed", "", "The bridge that answers already runs this choice — no restart was needed.")
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
            else:
                set_public_restart(cfg, "restarting", "restart", "Restarting the bridge…")
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
                schedule_bridge_restart()
        elif self.path == "/api/dhd/toggle":
            # The DHD master switch (build .74) — one click: the
            # choice is saved and the whole ladder runs in this
            # process (certificate, proxy, self-check) with no
            # bridge restart, because the proxy is threads, not the
            # bridge's own bind. The one exception: if the Local
            # server currently binds this bridge publicly, DHD mode
            # requires the bridge on loopback only, so enabling
            # records the choice and rides the exit-75 supervisor
            # restart — the relaunched bridge binds loopback and
            # brings the proxy up in main(). A failed step names
            # itself and leaves the previous state untouched.
            data = self._body()
            want = bool(data.get("enabled"))
            if not want:
                dhd_save(cfg, {"enabled": False})
                ensure_access_code(cfg)
                cfg = load_config()
                dhd_stop_proxy()
                DHD_STATE["last_error"] = ""
                self._send(200, {"ok": True, "dhd": dhd_state_payload(cfg, self._client_is_loopback())})
            else:
                # Enabling is an operation: the page and chat
                # render the same persisted step card while the
                # bridge runs prerequisites, certificate, proxy,
                # and reachability — no model work is involved.
                _op_id = dhd_start_enable_operation("page", str(data.get("sudo_password") or data.get("sudoPassword") or ""))
                self._send(200, {"ok": True, "operationId": _op_id, "operation": op_snapshot(_op_id), "dhd": dhd_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/dhd/install":
            # The safety-net install action: install the named
            # missing prerequisites (or every missing installable
            # one) through the detected distro family's package
            # step, then release any DHD enable operation waiting
            # on them so it continues automatically. An install
            # failure returns the step's exact manual commands.
            data = self._body()
            _pieces = data.get("pieces") if isinstance(data.get("pieces"), list) else None
            _result = dhd_install_prereqs(_pieces, str(data.get("sudo_password") or data.get("sudoPassword") or ""))
            _failed = [r for r in _result["results"] if not r["ok"]]
            if _failed:
                _commands = []
                for r in _failed:
                    _commands.extend(r.get("commands") or [])
                _detail = " ".join(r.get("detail", "") for r in _failed)
                _dhd_release_waiting_ops(False, _detail, _commands)
                self._send(200, {"ok": False, "install": _result, "commands": _commands, "dhd": dhd_state_payload(load_config(), self._client_is_loopback())})
            else:
                _dhd_release_waiting_ops(True, "Prerequisites installed.")
                _op_id = None
                _cfg_now = load_config()
                if dhd_settings(_cfg_now)["enabled"] and not DHD_STATE.get("running"):
                    _op_id = dhd_start_enable_operation("install", str(data.get("sudo_password") or data.get("sudoPassword") or ""))
                self._send(200, {"ok": True, "install": _result, "operationId": _op_id, "operation": op_snapshot(_op_id) if _op_id else op_latest("dhd-enable"), "dhd": dhd_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/dhd/domain":
            # The optional custom domain: free-form, saved in the
            # dhd block so it survives restarts and updates. When
            # DHD is live, saving a domain starts the certificate
            # step in the background (ACME can take a few seconds);
            # the door keeps answering on the current certificate
            # meanwhile and the state reads certificate-pending
            # until the real one is installed.
            data = self._body()
            domain = str(data.get("domain") or "").strip().lower()
            domain = domain.replace("https://", "").replace("http://", "").split("/")[0]
            dhd_save(cfg, {"domain": domain})
            cfg = load_config()
            if domain and dhd_settings(cfg)["enabled"] and DHD_STATE["running"]:
                threading.Thread(target=lambda: dhd_bring_up(load_config()), daemon=True).start()
            self._send(200, {"ok": True, "dhd": dhd_state_payload(cfg, self._client_is_loopback())})
        elif self.path == "/api/dhd/cert":
            # The certificate step on its own: obtain (or renew)
            # the saved domain's certificate now, via the built-in
            # ACME client, and reload the proxy with it.
            cfg_now = load_config()
            ds_now = dhd_settings(cfg_now)
            if not ds_now["domain"]:
                self._send(200, {"ok": False, "error": "No domain is saved yet - add one first.", "dhd": dhd_state_payload(cfg_now, self._client_is_loopback())})
            else:
                ok = dhd_bring_up(cfg_now)
                self._send(200, {"ok": ok, "dhd": dhd_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/settings/public/restart":
            # The row's Restart bridge control: the retry/fallback
            # for a saved choice that is not live (a failed restart,
            # or a pending state left by an older build). Same flow
            # as the toggle: record the request, exit 75, let the
            # supervisor relaunch, and let the new bridge verify.
            cfg_now = load_config()
            if not public_state_payload(cfg_now, True)["pending"]:
                self._send(200, {"ok": True, "publicServer": public_state_payload(cfg_now, self._client_is_loopback())})
            else:
                set_public_restart(cfg_now, "restarting", "restart", "Restarting the bridge…")
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
                schedule_bridge_restart()
        elif self.path == "/api/settings/public/regenerate":
            # A fresh code. The shared-code cookie stops matching
            # at once, and no new knock can use the old code —
            # but a device already approved keeps its OWN token
            # (Build 2): trusted devices are revoked one by one in
            # the Door list, not by changing the code under them.
            block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
            block = dict(block)
            block["access_code"] = generate_access_code()
            if "enabled" not in block:
                block["enabled"] = False
            cfg["public_server"] = block
            save_config(cfg)
            self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/settings/public/devices/approve":
            # Trusted-device approval (Build 2, #173): a pending
            # device gets its own token and 30 days. Served to any
            # session that is itself inside the door (loopback or
            # gate-accepted — the gate above has already judged);
            # a waiting device's poll picks the approval up.
            data = self._body()
            dev = door_approve(load_config(), str(data.get("id") or ""))
            if dev is None:
                self._send(200, {"ok": False, "error": "That device is not waiting anymore — its knock may have expired.", "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
            else:
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/settings/public/devices/deny":
            data = self._body()
            if door_deny(load_config(), str(data.get("id") or "")):
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
            else:
                self._send(200, {"ok": False, "error": "That device is not waiting anymore.", "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/settings/public/devices/revoke":
            # Per-device revoke: the record goes, and with it the
            # token — that device is at the gate on its next knock,
            # and no one else's session notices.
            data = self._body()
            if door_revoke(load_config(), str(data.get("id") or "")):
                self._send(200, {"ok": True, "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
            else:
                self._send(200, {"ok": False, "error": "That device is not in the list.", "publicServer": public_state_payload(load_config(), self._client_is_loopback())})
        elif self.path == "/api/shell":
            data = self._body()
            try:
                self._send(200, run_shell(
                    str(data.get("command", "")),
                    sudo=bool(data.get("sudo")),
                    sudo_password=str(data.get("sudo_password") or ""),
                ))
            except Exception as e:
                self._send(200, {"command": data.get("command", ""), "output": str(e), "exit_code": 1})
        elif self.path == "/api/seraph/act":
            data = self._body()
            self._send(200, seraph_act(str(data.get("action") or ""),
                                       data))
        elif self.path == "/api/shell/session/open":
            # CLI Mode (Build 3): open (or return) the chat's one
            # standing interactive session.
            data = self._body()
            sess = shell_session_open(str(data.get("chatId") or "main"),
                                      data.get("cols") or 80,
                                      data.get("rows") or 24)
            self._send(200, shell_session_state(sess.chat_id, 0))
        elif self.path == "/api/shell/session/input":
            data = self._body()
            ok = shell_session_input(str(data.get("chatId") or "main"),
                                     str(data.get("text") or ""),
                                     "operator",
                                     raw=bool(data.get("raw")))
            if ok:
                self._send(200, {"ok": True})
            else:
                self._send(200, {"ok": False, "error": "No live shell session for this chat — open one first."})
        elif self.path == "/api/shell/session/close":
            data = self._body()
            chat_id = str(data.get("chatId") or "main")
            shell_session_close(chat_id)
            self._send(200, shell_session_state(chat_id, 0))
        elif self.path == "/api/shell/session/resize":
            data = self._body()
            sess = shell_session_get(str(data.get("chatId") or "main"))
            if sess is not None:
                sess.resize(data.get("cols") or 80, data.get("rows") or 24)
            self._send(200, {"ok": sess is not None})
        elif self.path == "/api/update/apply":
            # Starts the end-to-end apply: the worker downloads,
            # verifies (gzip-tolerant), and stages the release
            # package under updates/<tag>/package, then a detached
            # helper runs the staged installer and relaunches the
            # app. Progress and outcome are read back through
            # GET /api/update/result.
            self._send(200, update_apply_payload(cfg))
        elif self.path == "/api/downloads/save":
            # The dashboard's download-save flow (build .48): the
            # bytes behind a download click inside the desktop shell
            # (chip .png, export ZIP), written into the operator's
            # Downloads folder under a sanitized name — the shell's
            # webview cannot save files the way a browser does.
            self._send(200, save_download_payload(self._body()))
        elif self.path == "/api/sudo/arm":
            # Sudo mode (Build 3): the operator arms it from
            # Settings. The password rides this one request into
            # "sudo -v" and is forgotten — never stored, logged or
            # synced. The reply carries the fresh state so the
            # card can show exactly what the bridge holds (the
            # switch gate's three-way contract).
            data = self._body()
            password = str(data.get("password") or "")
            if not password:
                self._send(200, {"ok": False, "error": "Enter the sudo password to arm sudo mode.", "sudo": sudo_state()})
            else:
                ok, err = sudo_arm(password)
                if ok:
                    self._send(200, {"ok": True, "sudo": sudo_state()})
                else:
                    self._send(200, {"ok": False, "error": "sudo mode was not armed: " + str(err), "sudo": sudo_state()})
        elif self.path == "/api/sudo/disarm":
            # Instant and total: the flag flips and the timestamp
            # dies with it (sudo_disarm runs sudo -k).
            self._send(200, {"ok": True, "sudo": sudo_disarm()})
        elif self.path == "/api/hardware/redetect":
            # Re-run profiling on the real machine and store it:
            # detected hardware, derived settings, reasoning, and
            # the log lines (also appended to hardware-profile.log).
            detected = detect_hardware()
            settings, tier, tier_label, reasoning = derive_hardware_settings(detected)
            now = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
            profile = {"detected": detected, "settings": settings, "autoSettings": settings,
                       "source": "auto", "tier": tier, "tierLabel": tier_label,
                       "reasoning": reasoning, "detectedAt": now,
                       "installLog": [f"[hardware] {now} — re-detect from Settings",
                                      f"[hardware] CPU: {detected['cpuModel']} — {detected['cpuCores']} logical cores",
                                      f"[hardware] RAM: {detected['totalRamGB']} GiB",
                                      f"[hardware] GPU: {detected['gpuName'] or 'none'} ({detected['gpuKind']})"] + [f"[hardware] → {r}" for r in reasoning]}
            save_hardware_profile(profile)
            self._send(200, {"ok": True, "profile": hardware_profile_view(load_config())})
        elif self.path == "/api/hardware/override":
            data = self._body()
            current = load_config().get("hardware_profile") or {}
            detected = current.get("detected") or detect_hardware()
            def _clamp(v, lo, hi, fallback):
                try:
                    return max(lo, min(hi, int(v)))
                except Exception:
                    return fallback
            auto = current.get("autoSettings") or current.get("settings") or {}
            settings = {
                "threads": _clamp(data.get("threads"), 1, 32, int(auto.get("threads") or 4)),
                "ctxSize": _clamp(data.get("ctxSize"), 512, 65536, int(auto.get("ctxSize") or 4096)),
                "batchSize": _clamp(data.get("batchSize"), 32, 4096, int(auto.get("batchSize") or 512)),
                "gpuLayers": _clamp(data.get("gpuLayers"), 0, 99, int(auto.get("gpuLayers") or 0)),
            }
            now = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
            profile = {"detected": detected, "settings": settings,
                       "autoSettings": auto if auto else settings, "source": "manual",
                       "tier": current.get("tier", "fluid"), "tierLabel": current.get("tierLabel", ""),
                       "reasoning": current.get("reasoning") or [], "detectedAt": current.get("detectedAt") or now,
                       "installLog": [f"[hardware] {now} — manual override in Settings: threads {settings['threads']}, ctx {settings['ctxSize']}, batch {settings['batchSize']}, gpu layers {settings['gpuLayers']}"]}
            save_hardware_profile(profile)
            self._send(200, {"ok": True, "profile": hardware_profile_view(load_config())})
        elif self.path == "/api/persona":
            data = self._body()
            os.makedirs(os.path.dirname(SOUL_PATH), exist_ok=True)
            if isinstance(data.get("soul"), str):
                with open(SOUL_PATH, "w") as f:
                    f.write(data["soul"])
            if isinstance(data.get("memory"), str):
                with open(MEMORY_PATH, "w") as f:
                    f.write(data["memory"])
            self._send(200, {"ok": True})
        elif self.path == "/api/llamafile/connect":
            endpoint = cfg.get("live", {}).get("llamafile_endpoint", "http://localhost:8080/v1")
            alive = llamafile_alive(endpoint)
            started = False
            error = None
            if not alive:
                if cfg.get("startup", "on-ui-launch") == "on-boot":
                    # Boot mode: the system service owns the server.
                    # The bridge probes and reports; it never spawns
                    # a second copy behind the service's back.
                    error = ("No answer from navi.llamafile at " + endpoint +
                             " — this install starts the server at system boot, "
                             "so the boot service should be running it; check that service.")
                else:
                    # App-load mode: connecting means starting the
                    # server, not just probing for it.
                    alive, error = start_llamafile(cfg, endpoint)
                    started = alive
            self._send(200, {
                "ok": alive,
                "status": "running" if alive else "stopped",
                "endpoint": endpoint,
                "started": started,
                "error": None if alive else error,
            })
        elif self.path == "/api/voice/mic-report":
            # Build 6 (#6, telemetry half): a window reports its
            # own mic reading (granted / refused / no recording
            # API) each time push-to-talk is attempted in it. The
            # report is logged in words and kept for /api/voice —
            # the app-window capture question (#6) has been open
            # since 2026-11-07 for want of exactly this reading
            # existing somewhere durable.
            data = self._body()
            st = str(data.get("state") or "")
            if st not in ("ok", "refused", "unavailable", "unknown"):
                self._send(200, {"ok": False, "error": "unknown mic state"})
            else:
                _MIC_REPORT["state"] = st
                _MIC_REPORT["at"] = int(time.time())
                print("Navi voice: a window reports its own mic as "
                      + {"ok": "granted", "refused": "refused",
                         "unavailable": "no recording API",
                         "unknown": "not tested"}[st]
                      + " (push-to-talk attempt).", flush=True)
                self._send(200, {"ok": True})
        elif self.path == "/api/voice/warmup":
            # The dashboard fires this when the Voice view opens:
            # load the speech engines then — while the operator is
            # looking at the voice panel — instead of inside Call,
            # where the load would stall the pickup, and never at
            # bridge boot, because on a small machine the models are
            # real RAM that chat shouldn't pay for uninvited. The
            # loaders cache in _voice_cache and never evict, so once
            # warm she stays warm for the life of the bridge.
            vc = voice_config(cfg)
            warmed = []
            errors = {}
            if not vc["enabled"]:
                errors["voice"] = "Voice is turned off."
            else:
                try:
                    _stt_model(vc)
                    warmed.append("stt")
                except Exception as e:
                    errors["stt"] = str(e)
                # Build 6 (#221): warming also proves the decode
                # path — the model can load perfectly over a
                # decoder that cannot decode (the av drift), and
                # the warmup answer must not call that warm.
                if "stt" not in errors:
                    dec_ok, dec_exc = _voice_decode_probe()
                    if dec_ok:
                        warmed.append("decode")
                    else:
                        errors["decode"] = _decode_detail(dec_exc)
                try:
                    if vc["tts_engine"] == "kokoro":
                        # v0.4.1e (#229): warmup runs the CUDA
                        # EXECUTION probe beside the decode probe,
                        # before the pipeline is built on its
                        # verdict — a fallen-back verdict is a
                        # working state with a named reason, not
                        # a warmup error.
                        _voice_cuda_probe()
                        _kokoro_pipeline(vc)
                    else:
                        _piper_load(vc)
                    warmed.append("tts")
                except Exception as e:
                    errors["tts"] = str(e)
            self._send(200, {"ok": not errors, "warmed": warmed, "errors": errors})
        elif self.path == "/api/voice/settings":
            # The Voice tab's settings, saved into navi.config.json.
            data = self._body()
            patch = {}
            if isinstance(data.get("enabled"), bool):
                patch["enabled"] = data["enabled"]
            if data.get("stt_model") in ("tiny.en", "base.en", "small.en"):
                patch["stt_model"] = data["stt_model"]
            if data.get("tts_engine") in ("kokoro", "piper"):
                patch["tts_engine"] = data["tts_engine"]
            if isinstance(data.get("tts_voice"), str) and data["tts_voice"].strip():
                patch["tts_voice"] = data["tts_voice"].strip()[:60]
            for key in ("rate", "pitch"):
                try:
                    val = float(data.get(key))
                    if 0.5 <= val <= 2.0:
                        patch[key] = val
                except (TypeError, ValueError):
                    pass
            cur = save_voice_config(patch)
            self._send(200, {"ok": True, "config": cur})
        elif self.path == "/api/voice/test":
            # "Hear her voice": synthesize one line and hand the WAV
            # to the dashboard to play. Honest failure when the
            # engine isn't installed — never a placeholder tone.
            data = self._body()
            vc = voice_config(cfg)
            line = str(data.get("text") or "").strip() or (
                "Hey — it's me, Navi. This is my voice, running entirely "
                "on your machine. Call me any time."
            )
            try:
                wav = synthesize_wav(line, vc)
                self._send(200, {
                    "ok": True,
                    "audio_b64": base64.b64encode(wav).decode(),
                    "mime": "audio/wav",
                })
            except VoiceUnavailable as e:
                self._send(200, {"ok": False, "error": str(e)})
            except Exception as e:
                self._send(200, {"ok": False, "error": "Speech failed: " + str(e)})
        elif self.path == "/api/voice/tts":
            # One line of hers in, the spoken WAV out — the call
            # panel plays her replies sentence by sentence.
            data = self._body()
            vc = voice_config(cfg)
            try:
                wav = synthesize_wav(str(data.get("text") or ""), vc)
                self._send(200, {
                    "ok": True,
                    "audio_b64": base64.b64encode(wav).decode(),
                    "mime": "audio/wav",
                })
            except VoiceUnavailable as e:
                self._send(200, {"ok": False, "error": str(e)})
            except Exception as e:
                self._send(200, {"ok": False, "error": "Speech failed: " + str(e)})
        elif self.path == "/api/voice/stt":
            # One push-to-talk clip in (base64; webm from the
            # browser's recorder, wav also fine — the decoder sniffs
            # the real format), its transcript out.
            data = self._body()
            vc = voice_config(cfg)
            if not vc["enabled"]:
                self._send(200, {"ok": False, "error": "Voice is turned off — enable it in the Voice tab."})
            else:
                try:
                    audio = base64.b64decode(str(data.get("audio_b64") or ""))
                except Exception:
                    audio = b""
                try:
                    text = transcribe_audio(audio, vc)
                    self._send(200, {"ok": True, "text": text})
                except VoiceUnavailable as e:
                    self._send(200, {"ok": False, "error": str(e)})
                except Exception as e:
                    self._send(200, {"ok": False, "error": "Transcription failed: " + str(e)})
        elif self.path == "/api/voice/call/start":
            # Opens the call record. The turns themselves travel
            # over /api/chat (same brain as typed chat) and are
            # recorded here, one by one, as they happen.
            vc = voice_config(cfg)
            if not vc["enabled"]:
                self._send(200, {"ok": False, "error": "Voice is turned off — enable it in the Voice tab."})
            else:
                state = load_voice_state()
                if not state.get("call"):
                    state["call"] = {
                        "id": str(int(time.time() * 1000)),
                        "started_at": time.strftime("%Y-%m-%dT%H:%M:%S%z"),
                        "turns": [],
                    }
                    save_voice_state(state)
                self._send(200, {"ok": True, "call": state["call"]})
        elif self.path == "/api/voice/call/turn":
            data = self._body()
            state = load_voice_state()
            call = state.get("call")
            if not call:
                self._send(200, {"ok": False, "error": "No call in progress."})
            else:
                turn = {
                    "who": "navi" if data.get("who") == "navi" else "you",
                    "text": str(data.get("text") or "")[:4000],
                    # Issue #116: when this turn was spoken (epoch
                    # seconds), so the transcript can carry the same
                    # timestamps the chat thread wears.
                    "ts": int(time.time()),
                }
                ex = data.get("exec")
                if isinstance(ex, dict):
                    turn["exec"] = {
                        "command": str(ex.get("command") or "")[:500],
                        "output": str(ex.get("output") or "")[:4000],
                    }
                call.setdefault("turns", []).append(turn)
                state["call"] = call
                save_voice_state(state)
                self._send(200, {"ok": True})
        elif self.path == "/api/voice/call/end":
            # The finished call joins the permanent record:
            # voice/calls.json, newest last; the dashboard files
            # the turns into the conversation itself.
            state = load_voice_state()
            call = state.get("call")
            if not call:
                self._send(200, {"ok": True, "call": None})
            else:
                call["ended_at"] = time.strftime("%Y-%m-%dT%H:%M:%S%z")
                calls = load_voice_calls()
                calls.append(call)
                os.makedirs(VOICE_DIR, exist_ok=True)
                with open(VOICE_CALLS_PATH, "w") as f:
                    json.dump(calls, f)
                state["call"] = None
                save_voice_state(state)
                self._send(200, {"ok": True, "call": call})
        elif self.path == "/api/notifications/read":
            # v0.4.7e (#242): the bell's write — {"ids": [...]} or
            # {"all": true} marks records read; unknown ids are
            # ignored, not errors. The answer is the fresh payload.
            data = self._body()
            self._send(200, notifications_mark_read(
                data.get("ids"), data.get("all") is True, cfg))
        elif self.path == "/api/push/subscribe":
            # v0.5.14e (#272): a phone subscribes to this
            # node's tray. The bridge validates the
            # subscription's keys before anything is stored;
            # a refusal is named and nothing is held.
            # v0.6.7e (#332): the caller's door-device identity
            # rides along when its token speaks for it, so the
            # record keys on the device, not the endpoint.
            data = self._body()
            _pdev = ""
            try:
                _pok, _pdev_rec, _pleg = door_gate_acceptance(
                    cfg, door_cookie_value(self.headers.get("Cookie")))
                if _pok and _pdev_rec is not None:
                    _pdev = str(_pdev_rec.get("id") or "")
            except Exception:
                _pdev = ""
            view, err = push_subscribe(data, cfg, device=_pdev)
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "subscription": view})
        elif self.path == "/api/push/remove":
            # v0.6.7e (#332): remove one subscription record
            # by id — from any seat. That record stops ringing;
            # the tray and every other record are untouched.
            data = self._body()
            self._send(200, push_remove_record(data.get("id"), cfg))
        elif self.path == "/api/push/unsubscribe":
            # v0.5.14e (#272): this device stops ringing. The
            # tray is untouched; only the door comes down.
            data = self._body()
            self._send(200, push_unsubscribe(data, cfg))
        elif self.path == "/api/chat/route":
            # v0.4.1e (#228): a route flip, validated against the
            # access the operator actually has (chat_route_set).
            # The answer carries the route the bridge stored; a
            # refusal carries its reason word and its sentence,
            # and the store is untouched — the pill degrades on
            # its own face, never a silent snap-back.
            data = self._body()
            _cid = str(data.get("chatId") or "")[:80]
            if not _cid:
                self._send(200, {"ok": False, "reason": "no-chat",
                                 "error": "That flip named no chat."})
                return
            _ans, _ref = chat_route_set(cfg, _cid, data.get("route"))
            if _ref is not None:
                self._send(200, {"ok": False, "reason": _ref[0],
                                 "error": _ref[1],
                                 "route": chat_route_get(_cid)})
            else:
                self._send(200, _ans)
        elif self.path == "/api/chat":
            data = self._body()
            messages = data.get("messages", [])
            # Sliding history: only the most recent exchanges ride
            # to the model. Prefill is paid per token on CPU, so an
            # ever-growing transcript makes every reply slower than
            # the last; the dashboard trims before sending and this
            # cap holds the line for any other caller. The full
            # conversation still lives in the dashboard — only the
            # model's view slides.
            if isinstance(messages, list) and len(messages) > CHAT_HISTORY_MESSAGES:
                messages = messages[-CHAT_HISTORY_MESSAGES:]
            # Citation footers never ride in message text (see
            # strip_citation_footer): scrub any an older build stored
            # in this thread's assistant history before it reaches
            # the model, so she never learns to imitate one.
            if isinstance(messages, list):
                messages = [
                    dict(m, content=strip_citation_footer(m.get("content")))
                    if isinstance(m, dict) and m.get("role") == "assistant" else m
                    for m in messages
                ]
            sudo = bool(data.get("sudo"))
            sudo_password = str(data.get("sudo_password") or "")
            chat_mode = data.get("mode") if data.get("mode") in ("cli", "navi") else None
            memory_mode = data.get("memory_mode") if data.get("memory_mode") in ("autonomous", "ask", "trigger") else "autonomous"
            last = messages[-1]["content"] if messages else ""
            endpoint = cfg.get("live", {}).get("llamafile_endpoint", "http://localhost:8080/v1")
            # Chip invocation: "/chip <name>", "run the <name> chip",
            # or a loaded chip's name on its own runs the chip right
            # here and its output lands in the conversation — local
            # code, so it works with or without the model connected.
            # A bare "/chip" names no chip: answer with the shelf
            # itself — every installed chip and a line on what it
            # does — so the operator can pick one. An empty shelf
            # says so plainly and points at where chips come from.
            # A bare "/rag" names no question: answer with the
            # packs themselves — what is installed and which packs
            # are active — mirroring bare "/chip" and its shelf.
            if (last or "").strip().lower() == "/rag":
                _packs = local_rag_packs()
                if _packs:
                    _lines = []
                    for p in _packs:
                        _lines.append(
                            "• " + p["name"] + " — " + str(p["docCount"])
                            + " documents · " + ("active" if p["active"] else "off")
                        )
                    _reply = (
                        "Knowledge packs on this machine — only the active ones feed my answers:\n"
                        + "\n".join(_lines)
                    )
                else:
                    _reply = (
                        "No knowledge packs are installed yet. Open the RAG tab and install one "
                        "from the GitHub catalog (topic: navi-rag) or forge your own in RAG Studio; "
                        "then my answers can draw on it, and I'll name the pack when one informs an answer."
                    )
                payload = {
                    "reply": _reply,
                    "exec": None,
                    "mode": "rag",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_reply, cfg)
                self._send(200, payload)
                return
            if (last or "").strip().lower() == "/chip":
                _reply = chip_shelf_reply("./")
                payload = {
                    "reply": _reply,
                    "exec": None,
                    "mode": "chip",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_reply, cfg)
                self._send(200, payload)
                return
            # "./name args..." runs a chip the way ./ runs a
            # program (0.1.9e, issue #97) — the composer's pill
            # shelf is retired, and this is the keyboard way in.
            # Bare "./" answers with the shelf. The latch decides
            # collisions: a $-forced line or a CLI-latched composer
            # never reaches here, so under the latch "./x" is still
            # the shell's own.
            _lt = (last or "").strip()
            if (
                _lt.startswith("./")
                and not _lt.startswith("$")
                and chat_mode != "cli"
            ):
                _rest = _lt[2:]
                if not _rest.strip():
                    _reply = chip_shelf_reply("./")
                    payload = {
                        "reply": _reply,
                        "exec": None,
                        "mode": "chip",
                        "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    }
                    if data.get("speak"):
                        payload["spoken"] = speak_reply(_reply, cfg)
                    self._send(200, payload)
                    return
                _chip, _cargs = resolve_chip_invocation(_rest, local_chips())
                if isinstance(_cargs, list):
                    if _cargs:
                        _reply = (
                            "That name fits more than one chip on this machine: "
                            + ", ".join(c["name"] for c in _cargs)
                            + " — say which one, as ./<name>."
                        )
                    else:
                        _reply = (
                            "No chip by that name is loaded on this machine. "
                            + chip_shelf_reply("./")
                        )
                    payload = {
                        "reply": _reply,
                        "exec": None,
                        "mode": "chip",
                        "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    }
                    if data.get("speak"):
                        payload["spoken"] = speak_reply(_reply, cfg)
                    self._send(200, payload)
                    return
                _result = execute_chip(_chip, _cargs, cfg)
                payload = {
                    "reply": "Slotted in " + _chip["name"] + " and ran it — here's what it gave me:",
                    "exec": _result,
                    "mode": "chip",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_result.get("output") or payload["reply"], cfg)
                self._send(200, payload)
                return
            _query = chip_invocation(last)
            if _query is not None:
                _chip, _cargs = match_local_chip(_query, local_chips())
                if _chip is not None:
                    _result = execute_chip(_chip, _cargs, cfg)
                    payload = {
                        "reply": "Slotted in " + _chip["name"] + " and ran it — here's what it gave me:",
                        "exec": _result,
                        "mode": "chip",
                        "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    }
                    if data.get("speak"):
                        payload["spoken"] = speak_reply(_result.get("output") or payload["reply"], cfg)
                    self._send(200, payload)
                    return
                if (last or "").strip().lower().startswith("/chip"):
                    _names = ", ".join(c["name"] for c in local_chips()) or "none"
                    self._send(200, {
                        "reply": "No chip by that name is loaded on this machine — the chips folder holds: " + _names + ".",
                        "exec": None,
                        "mode": "chip",
                        "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    })
                    return
            # DHD operations from chat: a direct request to turn
            # DHD on or off is deterministic bridge work, decided
            # before any model or CLI route. Enabling starts the
            # same operation the DHD page uses; the reply carries
            # its snapshot and the dashboard renders the live card
            # in this thread. No model generation happens while
            # the operation runs.
            _dhd_intent = dhd_chat_intent(last or "")
            if _dhd_intent == "enable":
                if dhd_settings(cfg)["enabled"] and DHD_STATE.get("running"):
                    _reply = _dhd_on_summary(dhd_state_payload(cfg, True))
                    _op = None
                else:
                    _op_id = dhd_start_enable_operation("chat", sudo_password)
                    _op = op_snapshot(_op_id)
                    _reply = "Starting DHD now. The card below is the live record — prerequisites, certificate, proxy, and the reachability check — and it stays in this thread when it finishes."
                payload = {
                    "reply": _reply, "exec": None, "mode": "operation",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    "operation": _op,
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_reply, cfg)
                self._send(200, payload)
                return
            if _dhd_intent == "disable":
                dhd_save(cfg, {"enabled": False})
                ensure_access_code(cfg)
                dhd_stop_proxy()
                DHD_STATE["last_error"] = ""
                _reply = "DHD is off. Apex answers only on this machine and your home network again; the domain and certificate stay saved for the next one-click start."
                payload = {
                    "reply": _reply, "exec": None, "mode": "operation",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    "operation": None,
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_reply, cfg)
                self._send(200, payload)
                return
            # CLI-first routing: with the model connected, a line
            # whose first word is a system command runs right here —
            # no model call, no waiting on her to orchestrate it —
            # and the raw output returns as the exec block the
            # dashboard files as a terminal block. A leading "$"
            # forces this route; a leading "> " forces the line to
            # Navi, and the marker is stripped before she sees it.
            _cmd, _navi_text = cli_route(last or "", chat_mode)
            if _navi_text is not None and _navi_text != (last or ""):
                if messages:
                    messages[-1] = dict(messages[-1], content=_navi_text)
                last = _navi_text
            if _cmd is not None and llamafile_alive(endpoint):
                try:
                    _result = run_shell(_cmd, sudo=sudo, sudo_password=sudo_password) if _cmd else {"command": "", "output": "", "exit_code": 0}
                except Exception as e:
                    _result = {"command": _cmd, "output": str(e), "exit_code": 1}
                payload = {
                    "reply": "",
                    "exec": _result,
                    "mode": "cli",
                    "llamafile": "running",
                }
                if data.get("speak") and _result.get("output"):
                    payload["spoken"] = speak_reply(_result["output"], cfg)
                self._send(200, payload)
                return
            # Memory from chat — the trigger (build .81, issue
            # #72): deterministic bridge work beside Vault filing,
            # decided before the model route and before the
            # command-line fallback, so it works with or without
            # the model connected. The fact rides the payload back
            # as data; the dashboard files it into her memory
            # store and draws the receipt in the thread. A
            # secret-shaped payload is refused in words and
            # nothing is written.
            _mtrig = memory_trigger(last or "")
            if _mtrig is not None:
                _mfact = ""
                if _mtrig[0] == "fact":
                    _mfact = normalize_memory_fact(_mtrig[1])
                else:
                    for _pm in reversed(messages[:-1]):
                        if isinstance(_pm, dict) and _pm.get("role") == "user" and (_pm.get("content") or "").strip():
                            _mfact = normalize_memory_fact(_pm.get("content"))
                            break
                _mlive = "running" if llamafile_alive(endpoint) else "stopped"
                if not _mfact:
                    _reply = "Point me at the thing itself — say it first and then \"remember this\", or say \"remember:\" and the fact, and I'll write it down word for word and show you the receipt."
                    payload = {"reply": _reply, "exec": None, "mode": "model", "llamafile": _mlive}
                elif memory_looks_secret(_mfact):
                    _reply = "I didn't save that — it looks like a secret, and secrets never go into my memory, not even when you ask me directly. If it needs keeping, it belongs somewhere built for secrets, not in the memory I carry into every conversation."
                    payload = {"reply": _reply, "exec": None, "mode": "model", "llamafile": _mlive, "memoryRefused": True}
                else:
                    _reply = "Written down. The receipt below is exactly what I saved — word for word — and Undo takes it back out if I got it wrong."
                    payload = {"reply": _reply, "exec": None, "mode": "model", "llamafile": _mlive, "memorySaves": [{"text": _mfact, "origin": "trigger", "key": memory_fact_key(_mfact), "proposed": False}]}
                    # v0.6.0e (#294): the save is recorded in
                    # the foundation store at the moment the
                    # bridge receipts it — provenance, super-
                    # session and the tombstone-grade record
                    # live there; the seat memory keeps its own
                    # path, untouched.
                    try:
                        daybook_record_capture(
                            _mfact, origin="trigger",
                            key=memory_fact_key(_mfact),
                            chat_id=str(data.get("chatId") or ""),
                            cfg=cfg)
                    except Exception:
                        pass
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_reply, cfg)
                self._send(200, payload)
                return
            _vfiling = re.search(r"(?:file|write|save|make|take)\s+(?:a\s+|this\s+)?note\b", (last or ""), re.I) or re.search(r"\bnote\s+(?:this|that)\s+down\b", (last or ""), re.I) or re.match(r"\s*note\s*:", (last or ""), re.I)
            if _vfiling:
                _vpath, _verr = vault_file_from_chat(last or "", messages)
                if _vpath:
                    _own = _vpath.startswith("Operator/")
                    payload = {
                        "reply": ("Filed it on your side of the desk — **" + _vpath + "** in the Vault. You can edit it there any time; I won't touch it again unless you ask.") if _own else ("Filed it on my side of the desk — **" + _vpath + "** in the Vault. It's mine to keep faithfully now: dated, in my own words, and I'll name it when it informs an answer."),
                        "exec": None,
                        "mode": "model",
                        "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                    }
                    if data.get("speak"):
                        payload["spoken"] = speak_reply(payload["reply"], cfg)
                    self._send(200, payload)
                    return
            # Case filing from chat (Build 12, #137): "file that as
            # a finding" / "file this to the case" lands in the OPEN
            # case, out loud — the deterministic sibling of Vault
            # filing, decided in the same place for the same reason:
            # it works with or without the model connected.
            # v0.5.15e (#291): the ask may name the destination —
            # "file this as a lead", "add that to the case as an
            # observable", "log it as an event" — and the named
            # part governs over every default. Spoken in the case's
            # own room with a part named, the words file the reply
            # they point at (the room's latest Navi reply) through
            # the same composition the tap uses, and the reply here
            # is its receipt, naming the landing. Anywhere else,
            # the words file the exchange's context into the open
            # case at the named part. A named place that is not a
            # filing part is refused by name, out loud, and
            # nothing is filed.
            _cfiling = re.search(r"\bfile\s+(?:this|that|it)\s+(?:as\s+(?:a\s+|an\s+)?(?:finding|lead|observable|entity|event|evidence|task|timeline|note|report)|to\s+the\s+case|into\s+the\s+case)\b", (last or ""), re.I) or re.search(r"\badd\s+(?:this|that|it)\s+(?:to|into)\s+the\s+case\b", (last or ""), re.I) or re.search(r"\badd\s+(?:this|that|it)\s+as\s+(?:a\s+|an\s+)?(?:finding|lead|observable|entity|event|evidence|task|timeline|note|report)\b", (last or ""), re.I) or re.search(r"\b(?:log|put)\s+(?:this|that|it)\s+(?:as\s+(?:a\s+|an\s+)?(?:finding|lead|observable|entity|event|evidence|task|timeline|note|report)|in\s+the\s+(?:observables|entities|findings|leads|events))\b", (last or ""), re.I)
            if _cfiling:
                _cpart, _cnonpart = case_named_part(last or "")
                _cfiled, _cerr = None, None
                if _cnonpart:
                    _cerr = case_part_refusal(_cnonpart)
                elif _cpart:
                    _cid = data.get("chatId")
                    _brec = None
                    if _cid is not None:
                        for _r in chats_all():
                            if str(_r.get("id")) == str(_cid):
                                _brec = _r
                                break
                    if _brec is not None and _brec.get("case"):
                        _bslug = str(_brec.get("case") or "")
                        _bmeta = _case_manifest(_bslug)
                        _blast = ""
                        for _m in reversed(_brec.get("messages") or []):
                            if (isinstance(_m, dict)
                                    and _m.get("who") == "NAVI"
                                    and not _m.get("caseFiledReply")
                                    and not _m.get("caseStarted")
                                    and str(_m.get("text") or "").strip()):
                                _blast = str(_m.get("text"))
                                break
                        _bbody = case_strip_machinery(_blast).strip()
                        if _bmeta is not None and _bbody:
                            _bf, _be = _case_reply_filing(
                                _bslug,
                                str(_bmeta.get("title") or _bslug),
                                _brec.get("id"), _bbody, _cpart,
                                "words")
                            if _bf:
                                _cfiled = {"case": str(_bmeta.get("title") or _bslug),
                                           "path": _bf["path"],
                                           "part": _cpart,
                                           "room": True}
                            else:
                                _cerr = _be
                if _cfiled is None and _cerr is None:
                    _cfiled, _cerr = case_file_from_chat(last or "", messages, cfg)
                if _cfiled:
                    if _cfiled.get("room"):
                        _creply = "Filed that reply into the case **" + _cfiled["case"] + "** — as " + _case_part_article(_cfiled["part"]) + " " + _cfiled["part"] + ", at `" + _cfiled["path"] + "`. It stands exactly as written, in my name; the landing is named here so a wrong part would show now, not at read-back."
                    elif _cfiled.get("part", "finding") == "finding":
                        _creply = "Filed it to the case **" + _cfiled["case"] + "** — a new finding at `" + _cfiled["path"] + "`, where it stays in plain markdown on the desk. Nothing filed silently, ever: that's the whole entry, and it's yours to edit in Case Files."
                    else:
                        _creply = "Filed it to the case **" + _cfiled["case"] + "** — as " + _case_part_article(_cfiled["part"]) + " " + _cfiled["part"] + ", at `" + _cfiled["path"] + "`, where it stays in plain markdown on the desk. Nothing filed silently, ever: that's the whole entry, and it's yours to edit in Case Files."
                else:
                    _creply = _cerr
                payload = {
                    "reply": _creply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_creply, cfg)
                self._send(200, payload)
                return
            # The Dossier (v0.5.6e, #260): "dossier: person …",
            # "sweep the handle …", "look up <email>", "look up
            # the number …", "find the person …" — the same
            # deterministic discipline as the routes below: the
            # bridge machinery runs, the run lands in the
            # Dossier's history, and the reply carries the
            # result plus the open dossier's state. Navi is
            # the second door to the same room (#241 pattern).
            _dzreply = None
            try:
                _dzreply = dossier_delete_chat_reply(last or "")
            except Exception:
                _dzreply = None
            if not _dzreply:
                try:
                    _dzreply = dossier_chat_reply(last or "")
                except Exception:
                    _dzreply = None
            if _dzreply:
                payload = {
                    "reply": _dzreply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_dzreply, cfg)
                self._send(200, payload)
                return
            # Pulse, spoken (v0.5.13e, #287): "keep a pulse on
            # X" and its pause/resume/remove kin route
            # deterministically — the watch is bridge state, so
            # the bridge answers from what it actually did.
            _preply = None
            try:
                _preply = pulse_chat_reply(last or "", cfg)
            except Exception:
                _preply = None
            if _preply:
                payload = {
                    "reply": _preply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_preply, cfg)
                self._send(200, payload)
                return
            # The case suite, spoken (v0.6.6e, #330): inside a
            # case room, "brief me" means the CASE — a fresh
            # briefing from its live state — and anywhere else,
            # "start an investigation on <that story>" is a
            # case birth from the Wire. Both are bridge state,
            # so the bridge answers from what it actually did.
            # These stand BEFORE the day route: the day's own
            # "brief me" keeps every non-case room.
            _csreply = None
            try:
                _csreply = case_brief_chat_reply(
                    str(data.get("chatId") or ""), last or "")
                if _csreply is None:
                    _csreply = case_birth_chat_reply(
                        str(data.get("chatId") or ""), last or "")
            except Exception:
                _csreply = None
            if _csreply:
                payload = {
                    "reply": _csreply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_csreply, cfg)
                self._send(200, payload)
                return
            # Her working day, spoken (v0.6.0e, #294/#295/
            # #296): "remind me …", "standing order: …",
            # "brief me", "recall …" route deterministically —
            # the day is bridge state, so the bridge answers
            # from what it actually did.
            _dyreply = None
            try:
                _dyreply = day_chat_reply(last or "", cfg,
                                          chat_id=str(data.get("chatId") or ""))
            except Exception:
                _dyreply = None
            if _dyreply:
                payload = {
                    "reply": _dyreply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_dyreply, cfg)
                self._send(200, payload)
                return
            # Fleet Answers (v0.6.1e, #293): an explicit fleet
            # trigger, or a plainly multi-part ask while this
            # node is in the fleet, routes to the fleet
            # machinery — decompose, fan out whole sub-tasks,
            # compose with the seams showing. A plain
            # single-part question returns None here and falls
            # through to the ordinary answer untouched (#289).
            _freply = None
            try:
                _freply = fleet_chat_reply(last or "", cfg,
                                           chat_id=str(data.get("chatId") or ""))
            except Exception:
                _freply = None
            if _freply:
                payload = {
                    "reply": _freply["reply"],
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if _freply.get("fleet"):
                    payload["fleet"] = _freply["fleet"]
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_freply["reply"], cfg)
                self._send(200, payload)
                return
            # Deep search + the conversational pull (v0.5.10e,
            # #282/#283): past the catalog, into the sources
            # themselves — a named source or a named hit routes
            # deterministically, an unnamed "dig deeper" is
            # proposed in the open and nothing is fetched; and
            # "pull that KEV card in", spoken in a case chat,
            # lands the card on that case's reference shelf,
            # receipted from what the bridge did. Both run
            # ahead of the #259 catalog routes so a live ask is
            # never answered from the mirror by mistake.
            _dreply = None
            try:
                _dreply = deep_search_chat_reply(last or "")
            except Exception:
                _dreply = None
            if not _dreply:
                try:
                    _dreply = wire_pull_chat_reply(last or "", data.get("chatId"), cfg)
                except Exception:
                    _dreply = None
            if _dreply:
                payload = {
                    "reply": _dreply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_dreply, cfg)
                self._send(200, payload)
                return
            # Catalog search + reference reading (v0.5.2e,
            # #259): the same deterministic discipline as the
            # lookup below — answered straight from the local
            # store (and, for references, from the record's own
            # links, read on demand and cited to the link), the
            # catalog's stamp on the answer's face, absence
            # stated as absence. "search the KEV for OpenSSH",
            # "any CVEs for X this week", "what CVEs affect X",
            # "read the references for CVE-…".
            _rm = re.search(r"\b(?:read|follow|check|open)\s+the\s+(?:references|links|refs)\b[^.]*?\b(CVE-\d{4}-\d{4,7})\b", last or "", re.I) or re.search(r"\bwhat\s+do\s+the\s+(?:references|links)\b[^.]*?\b(CVE-\d{4}-\d{4,7})\b", last or "", re.I)
            _sm = None
            if not _rm:
                _sm = (re.search(r"\bsearch\s+(?:the\s+)?(kev|catalog|store|center|cves?|vulnerabilit\w*)\s+for\s+(.+)", last or "", re.I)
                       or re.search(r"\bany\s+(cves?|kev(?:\s+entries)?|known[-\s]exploited(?:\s+entries)?)\s+for\s+(.+)", last or "", re.I)
                       or re.search(r"\bwhat\s+(cves?|vulnerabilit\w*)\s+(?:affect|hit)\s+(.+)", last or "", re.I))
            if _rm or _sm:
                try:
                    if _rm:
                        _ireply = INTEL.references_text(_rm.group(1).upper())
                    else:
                        _noun = (_sm.group(1) or "").lower()
                        _q = (_sm.group(2) or "").strip().rstrip("?").strip()
                        _kev_only = ("kev" in _noun) or ("known" in _noun)
                        _matched_only = bool(re.search(r"\bmatched\b|\bmy\s+stack\b", last or "", re.I))
                        _days = None
                        _dm = re.search(r"\blast\s+(\d+)\s+days?\b", last or "", re.I)
                        if _dm:
                            _days = int(_dm.group(1))
                            _q = re.sub(r"\s*last\s+\d+\s+days?\b", "", _q, flags=re.I).strip()
                        elif re.search(r"\bthis\s+week\b", last or "", re.I):
                            _days = 7
                            _q = re.sub(r"\s*this\s+week\b", "", _q, flags=re.I).strip()
                        elif re.search(r"\bthis\s+month\b", last or "", re.I):
                            _days = 30
                            _q = re.sub(r"\s*this\s+month\b", "", _q, flags=re.I).strip()
                        elif re.search(r"\btoday\b", last or "", re.I):
                            _days = 1
                            _q = re.sub(r"\s*today\b", "", _q, flags=re.I).strip()
                        _ireply = INTEL.search_text(_q, kev_only=_kev_only,
                                                    matched_only=_matched_only,
                                                    days=_days)
                except Exception:
                    _ireply = "The intel store could not be read just now — the Center's page (Apex | Apps → The Wire) shows its state."
                payload = {
                    "reply": _ireply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_ireply, cfg)
                self._send(200, payload)
                return
            # Threat intel lookup (Build 15, #165): "intel: …" or
            # "what do we know about CVE-…" is answered straight
            # from the local store by the same deterministic logic
            # as the routes above — no model round-trip, and the
            # store's own words about its freshness ride along in
            # the answer (absence is stated as absence).
            _im = re.match(r"\s*intel\s*[:\-]\s*(.+)", last or "", re.I)
            _cm = re.search(r"\bwhat do we know about\s+(CVE-\d{4}-\d{4,7})\b", last or "", re.I) or re.search(r"\blook\s*up\s+(CVE-\d{4}-\d{4,7})\b", last or "", re.I)
            if _im or _cm:
                _iquery = (_im.group(1) if _im else _cm.group(1)).strip()
                try:
                    _ireply = INTEL.lookup(_iquery)
                except Exception:
                    _ireply = "The intel store could not be read just now — the Center's page (Apex | Apps → The Wire) shows its state."
                payload = {
                    "reply": _ireply,
                    "exec": None,
                    "mode": "model",
                    "llamafile": "running" if llamafile_alive(endpoint) else "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(_ireply, cfg)
                self._send(200, payload)
                return
            if not llamafile_alive(endpoint):
                # Command-line mode: no model is connected, so the
                # message itself is the command. A leading "$" is
                # decoration and gets stripped.
                cmd = (last or "").strip()
                if cmd.startswith("$"):
                    cmd = cmd[1:].strip()
                if cmd == ">" or cmd.startswith("> "):
                    # The force-Navi marker, with no model to force
                    # to: strip it rather than hand the shell a bare
                    # redirection operator.
                    cmd = cmd[1:].strip()
                try:
                    result = run_shell(cmd, sudo=sudo, sudo_password=sudo_password) if cmd else {"command": "", "output": "", "exit_code": 0}
                except Exception as e:
                    result = {"command": cmd, "output": str(e), "exit_code": 1}
                payload = {
                    "reply": "(command-line mode — navi.llamafile is not connected)",
                    "exec": result,
                    "mode": "command-line",
                    "llamafile": "stopped",
                }
                if data.get("speak"):
                    payload["spoken"] = speak_reply(result["output"] or payload["reply"], cfg)
                self._send(200, payload)
                return
            exec_block = None
            m = LIST_INTENT.search(last or "")
            if m:
                # Navi's own initiative — when this runs elevated
                # it lands in the sudo audit (Build 3, #209).
                result = run_shell("ls " + expand(m.group(1)), sudo=sudo, sudo_password=sudo_password,
                                   chat_id=data.get("chatId"), audit=True)
                exec_block = result
                messages = messages + [{
                    "role": "system",
                    "content": "The shell was run for the user. Command: "
                    + result["command"] + "\nReal output:\n" + result["output"]
                    + "\nPresent this output accurately.",
                }]
            # Knowledge packs: only active packs feed her answers.
            # When a pack's material answers the question it rides
            # along as context; the sources themselves travel back
            # as data (the payload's "rag" field, below) — never as
            # words in her reply.
            _hits = rag_retrieve(last)
            _vhits = vault_search(last or "", 2)
            _vmentions = bool(re.search(r"\b(vault|notes?)\b", (last or ""), re.I))
            if _vhits and not _vmentions and _vhits[0]["score"] < 6:
                _vhits = []
            if _vhits:
                _vctx = "\n\n".join("[Vault · " + h["path"] + "] " + h["excerpt"] for h in _vhits)
                messages = messages + [{
                    "role": "system", "_budget_name": "rag",
                    "content": "Material from the shared Vault — the operator's and your own notes. Use it where it answers the question, in your own words. Do not name or cite the sources in your reply — the dashboard shows them under your message on its own:\n" + _vctx,
                }]
            if _hits:
                _ctx = "\n\n".join(
                    "[" + h["pack"] + " · " + h["document"] + "] " + h["text"]
                    for h in _hits
                )
                messages = messages + [{
                    "role": "system", "_budget_name": "rag",
                    "content": "Material from the operator's active knowledge packs — use it where it answers the question, in your own words. Do not name or cite the sources in your reply — the dashboard shows them under your message on its own:\n" + _ctx,
                }]
            # Navi's persona (Soul + Memory, from the config files)
            # rides along as the system prompt on every chat.
            #
            # v0.6.7e (#310): the prompt is assembled as NAMED
            # SEGMENTS now, and the local budget
            # (prompt_budget_apply) judges the whole before any
            # model sees it — shedding background in a fixed
            # order when the seat's context is small, and
            # refusing by name when even the standing core does
            # not fit, instead of dying at the model. Two of the
            # teachings are conditional from this build: the
            # shell teaching rides only a machine-directed turn
            # (the #240 gate's own read), the case teaching only
            # a case-shaped one. What was not carried is named
            # in the budget report on the payload.
            _persona_text = persona_prompt(cfg, str(data.get("operator_name") or ""), memory_mode)
            _segs = [{"name": "persona", "text": _persona_text,
                      "shed": None, "cap": False}]
            _not_carried = []

            def _seg(name, text, shed=None, cap=False):
                if text:
                    _segs.append({"name": name, "text": text,
                                  "shed": shed, "cap": cap})
            # The case this chat is bound to, read early: the
            # case teaching's condition and the case room's
            # blocks both stand on it.
            _bound_case = ""
            try:
                _cid = data.get("chatId")
                if _cid is not None:
                    for _rec in chats_all():
                        if str(_rec.get("id")) == str(_cid) and _rec.get("case"):
                            _bound_case = str(_rec["case"])
                            break
            except Exception:
                _bound_case = ""
            _open_case = open_case_payload(cfg)
            # Guardian (Build 14, #156): she answers questions
            # about this machine from what is true of it NOW, so
            # the Guardian brief — pressure, headroom, current
            # offenders, the latest actions — rides the system
            # prompt actually sent, exactly as assembled here
            # (the Mesh borrow path below carries a hygienic
            # copy of these same assembled messages — v0.6.7e,
            # #314/#315).
            try:
                if guardian_settings(cfg)["enabled"]:
                    _gtext = ("[Guardian — this machine right now]\n" + GUARDIAN.context_block()
                              + "\n(The Guardian brief above is background for questions "
                              "about this machine — answer from it in your own words, "
                              "and never read it back as your reply.)")
                    _seg("guardian", _gtext, shed=3, cap=True)
            except Exception:
                pass
            # v0.4.6e (#187): her own vitals ride beside the
            # Guardian brief — the teaching always (she may not
            # invent her speed), the measured figures when the
            # bridge's stopwatch has any.
            try:
                _seg("vitals-teaching", VITALS_TEACHING)
                _vitals = model_vitals_block()
                if _vitals:
                    _seg("vitals", _vitals, shed=4, cap=True)
            except Exception:
                pass
            # Case Files (Build 15, #161): while a case is open
            # she knows which one it is, and she can file her own
            # findings into it — her words, her name on them,
            # receipted in the thread (extract_finding_markers,
            # below, beside her memory saves).
            _case_open_text = ""
            if _open_case:
                _case_open_text = (
                    "[Case open — " + _open_case["name"] + "]\n"
                    "The case \"" + _open_case["name"] + "\" is open on the desk, and work in this conversation may belong to it. "
                    "When you reach a conclusion about the case that belongs in its record — a finding, in your own words — you may file it yourself by putting a line of exactly this form FIRST in your reply, before your words (a long reply can be cut off before it ends, and a marker that never lands files nothing): "
                    "[[finding: <short title> | <the finding, stated plainly>]] — optionally with a severity: [[finding[high]: <title> | <finding>]]. "
                    "At most two per reply, and only for real conclusions about this case — not small talk, and not the operator's words restated as yours. "
                    "The marker line is removed before the operator sees your reply; the filing is made in your name, a line goes into the case's timeline, and the operator is shown a receipt of exactly what you filed — never mention the marker itself."
                )
                _seg("case-open", _case_open_text)
            # Threat Intelligence Center (Build 15, #165): the
            # standing brief rides the system prompt the way the
            # Guardian's does — one bounded block (~700 tokens),
            # only when the store holds anything at all, always
            # carrying its freshness so she never states a stale
            # picture as a fresh one. In a case-bound chat the
            # case's own matched records ride too — the join
            # between the Center and the case on the desk.
            _ibrief_text = ""
            try:
                _ibrief_text = INTEL.brief_block()
                if _ibrief_text:
                    _seg("intel", _ibrief_text, shed=6, cap=True)
            except Exception:
                _ibrief_text = ""
            # Awareness (0.2.0e Build 7, #188/#189/Add.5): the
            # wire, the CTF board and the bounty board ride the
            # same way - one composed block, staleness included,
            # so "what's the sitch" is a behavior, not a search.
            try:
                _aw = _awareness_brief_block()
                if _aw:
                    _seg("awareness", _aw, shed=2, cap=True)
            except Exception:
                pass
            # CLI Mode (Build 3, #209): the standing teaching —
            # v0.6.7e (#310): it rides only a machine-directed
            # turn, judged by the #240 gate's own read (the
            # composer latched to CLI mode, a live session in
            # this chat, or the operator's words carrying
            # machine intent). On a conversation turn it is
            # weight the prompt does not carry.
            try:
                _machine_turn = shell_turn_machine_directed(
                    last or "", data.get("chatId"), chat_mode)
            except Exception:
                _machine_turn = True
            if _machine_turn:
                _seg("shell-teaching", SHELL_TEACHING)
            else:
                _not_carried.append("shell-teaching")
            # v0.4.5e (#241): the case-intake teaching —
            # v0.6.7e (#310): it rides a case-shaped turn (a
            # case bound to this chat or standing open, or the
            # operator's words naming case work).
            if _bound_case or _open_case or re.search(
                    r"\b(case|investigation|filing|file this)\b",
                    (last or ""), re.I):
                _seg("case-teaching", CASE_TEACHING)
            else:
                _not_carried.append("case-teaching")
            # v0.4.9e (#248): the Row rides the same way — the
            # teaching always, then the record's blocks: the
            # standing brief, the intent's supply, and the
            # pace block when her measured pace is slow.
            try:
                _seg("row-teaching", ROW_TEACHING)
                _rowblk = row_context_blocks(last or "")
                if _rowblk:
                    _seg("row", _rowblk, shed=1, cap=True)
                if "dossier" in (last or "").lower():
                    _dzblk = dossier_context_block()
                    if _dzblk:
                        _seg("dossier", _dzblk, shed=5, cap=True)
            except Exception:
                pass
            try:
                _sbrief = shell_session_brief(data.get("chatId"))
                if _sbrief:
                    _seg("shell-session", _sbrief, shed=10, cap=True)
            except Exception:
                pass
            if _bound_case:
                try:
                    _cterms = [t["term"] for t in _intel_case_terms(cfg, _bound_case)]
                    _cctx = INTEL.case_context(_cterms)
                    if _cctx:
                        _seg("case-ctx", _cctx, shed=7, cap=True)
                except Exception:
                    pass
                # Build 2 (#199): in the case's own room she
                # works the file, not just the chat — the case
                # brief (its contents WITH their review states)
                # rides along. v0.5.12e (#286): her answers in
                # the room are conversation, not filings — the
                # print-and-ask teaching below says so.
                try:
                    _cbrief = _case_brief_block(_bound_case)
                    if _cbrief:
                        _seg("case-brief", _cbrief, shed=9, cap=True)
                    # v0.5.10e (#283): the case's reference
                    # shelf rides the same way — listed by
                    # title and link, with the pull marker she
                    # may open a reply with to have a card's
                    # full text placed before her.
                    _shelfblk = case_shelf_block(_bound_case)
                    if _shelfblk:
                        _seg("shelf", _shelfblk, shed=8, cap=True)
                    _seg("working-case", (
                        "[Working this case]\n"
                        "This conversation is the case room for the case above. Your answers here are conversation, not filings: nothing you write is filed into the case automatically, whatever shape it takes — there is no block, no format, and no filing hidden in a reply. Answer fully and plainly, in whatever form serves the question.\n"
                        "Filing is the operator's act: every reply of yours in this room carries a \"File into this case\" button beneath it whose label names the part it will file to — the case has nine parts (v0.6.6e, #326): finding, lead, observable, entity, event, task, timeline, evidence, report — and the label is sensed from the reply's shape (v0.6.6e, #327): \"File into this case — as a finding\", \"— as tasks (3)\" for a next-step list, \"— as a timeline entry\" for a dated narrative, \"— as evidence\", \"— as report material\". Pressing it files that reply exactly as printed, to that part, in your name, receipted in the room with the landing named. You may suggest filing — \"that one is worth filing — as a lead\" — suggesting the part the content plainly is (a sourced claim is a lead, not a finding; a named person or system an entity; a bare indicator an observable; a thing that happened an event; a list of next steps tasks; a dated narrative the timeline; material for the write-up the report), and then let it stand: never wait on the press, never repeat the ask, and never narrate a filing that has not been receipted. The operator's words naming a part govern over the button's suggestion — absolutely, on every door. When the operator says \"brief me\" in this room, the bridge posts a fresh briefing drawn from the case's live state — that is bridge work, not yours to compose. If the conversation moves on, it moves on; an unpressed reply simply stays conversation.\n"
                        "Treat unverified material in the case brief above as leads: reason from them, never cite them as established. A detail that belongs to a DIFFERENT case is never written across — mention it in conversation instead. Facts drawn from a Wire record keep the record's values — its CVE id, CVSS, EPSS and dates; a reply the operator files that contradicts the record is flagged on its filing's receipt."
                    ))
                except Exception:
                    pass
            # The message seam: an exact CVE ID or a stack/case
            # term in the operator's own words pulls the record in
            # for THIS reply only.
            try:
                _ictx = INTEL.message_context(last or "")
                if _ictx:
                    messages = messages + [{"role": "system", "content": _ictx,
                                            "_budget_name": "intel-seam"}]
            except Exception:
                pass
            # v0.5.10e (#283): the operator names a shelf
            # reference for this reply — its full text rides in
            # for THIS reply only, cited as reference.
            if _bound_case:
                try:
                    for _nr in case_shelf_named_refs(_bound_case, last or ""):
                        messages = messages + [{
                            "role": "system",
                            "content": "[Reference " + _nr["id"] + " from this "
                                       "case's shelf — " + (_nr.get("title") or "")
                                       + " (" + (_nr.get("link") or "") + ")]\n"
                                       + (_nr.get("text") or "")
                                       + "\nThe operator named this reference "
                                       "for this reply: use it where it bears, "
                                       "and cite it as reference.",
                            "_budget_name": "shelf-ref",
                        }]
                except Exception:
                    pass
            # The budget itself (#310): the assembled prompt is
            # measured against this seat's context before any
            # model — local or lender — is asked. Overflow past
            # every shed is a named refusal, never a dead call.
            try:
                _ctx_size = runtime_settings(cfg)["ctx_size"]
            except Exception:
                _ctx_size = 4096
            messages, _budget_report = prompt_budget_apply(
                _segs, messages, _ctx_size)
            _budget_report["notCarried"] = _not_carried
            if _budget_report["overflow"]:
                self._send(200, {
                    "reply": ("This conversation's standing prompt — her persona and the briefs "
                              "that ride it — comes to about " + str(_budget_report["estTokens"])
                              + " tokens, and this seat's context is " + str(_budget_report["ctxSize"])
                              + ": even with every background brief shed, it does not fit, so "
                              "nothing was sent to the model. Raise the context size in "
                              "Settings → Runtime, or trim what stands in Soul and Memory, "
                              "and ask again."),
                    "exec": exec_block, "mode": "model",
                    "llamafile": "running",
                    "promptBudget": _budget_report})
                return
            try:
                model = cfg.get("live", {}).get("model_name") or "navi"
                # Navi Mesh (Ring 1): the model route is the ONLY
                # route that borrows — CLI mode, chips, and every
                # deterministic route above never do. The exact
                # assembled messages ride (persona/Vault/RAG/history
                # assembly stays borrower-side), and everything
                # below — citation strip, memory markers, speech —
                # stays borrower-side on the returned text,
                # unchanged. meshLocal is the per-request escape:
                # when the operator set it, no borrow is attempted
                # at all.
                _mesh_payload = None
                reply = None
                # v0.4.1e (#228): the route that governs is the
                # bridge's STORED route for this chat — the same
                # store the composer's pill renders from, so the
                # face and the road cannot disagree. meshLocal
                # stays honored as a per-request Local from an
                # older page caught mid-update.
                _route = chat_route_get(str(data.get("chatId") or ""))
                if data.get("meshLocal"):
                    _route = {"mode": "local"}
                if not (_route and _route.get("mode") == "local"):
                    _ms = mesh_settings(cfg)
                    if _ms["enabled"] and _ms["borrow"]["enabled"] and _ms["borrow"]["peers"]:
                        # v0.6.7e (#314/#315): what LEAVES for
                        # the lender is the hygienic copy — no
                        # marker teaching, no shell teaching, the
                        # intel brief in its for-share shape. The
                        # local assembly above is untouched; if
                        # no lender answers, it is what the local
                        # model sees, whole.
                        try:
                            _shared_brief = INTEL.brief_block(for_share=True)
                        except Exception:
                            _shared_brief = ""
                        reply, _mesh_payload = mesh_try_borrow(
                            borrow_hygiene_messages(
                                messages, _persona_text,
                                persona_prompt(cfg, str(data.get("operator_name") or ""), "trigger"),
                                _case_open_text, _ibrief_text, _shared_brief),
                            cfg, _ms, route=_route)
                if reply is None:
                    # Build 8 (#154): owner preemption — the
                    # owner's own work starting takes the machine
                    # from any FEDERATION borrow in flight (the
                    # borrower is answered "preempted", by name).
                    # Pairwise borrows are never preempted; local
                    # work simply takes the next slot for them.
                    mesh_preempt_borrowed()
                    with MESH_LOCK:
                        MESH_STATE["local_gen"] += 1
                    try:
                        reply = llamafile_chat(messages, endpoint, model)
                    finally:
                        with MESH_LOCK:
                            MESH_STATE["local_gen"] = max(0, MESH_STATE["local_gen"] - 1)
                            MESH_STATE["last_local_done"] = time.time()
                # The reply text stays citation-free (see
                # strip_citation_footer): sources are data, drawn
                # as pills under this one message by the dashboard.
                # Only a turn whose question actually retrieved pack
                # material carries any — once the conversation moves
                # on, there is nothing to show.
                reply = strip_citation_footer(reply)
                # v0.5.10e (#283): her reference pull — a reply
                # opening with [[pull-ref: R…]] has the named
                # cards' full texts placed before her and is
                # re-asked once, in this same exchange; the
                # pulls ride the payload back as data.
                _refpulls = []
                if _bound_case:
                    try:
                        reply, _refpulls, _mesh_payload = extract_pull_refs(
                            reply, _bound_case, messages, endpoint, model,
                            _mesh_payload)
                    except Exception:
                        _refpulls = []
                # v0.6.7e (#315): provenance decides what this
                # reply may DO here. A reply served by a Mesh
                # lender was not written by this node's Navi:
                # trust to answer is not trust to actuate. Its
                # memory markers are held as PROPOSALS (never
                # autonomous saves), its finding markers are
                # stripped without filing, and the shell gate
                # below is shut for it outright. A pull-ref
                # re-ask is this node's own generation — when
                # one fired, the words are local again.
                _borrowed = bool(_mesh_payload
                                 and _mesh_payload.get("servedBy") == "mesh"
                                 and not _refpulls)
                # Memory from chat — her own call (build .81,
                # #72): her [[remember: …]] markers come out of
                # the reply before it is shown or spoken, and the
                # saves ride the payload back as data for the
                # dashboard to file and receipt.
                reply, _msaves, _mrefused = extract_memory_markers(
                    reply, "ask" if _borrowed else memory_mode)
                # v0.6.0e (#294): her autonomous saves are
                # recorded in the foundation store as they are
                # receipted — the same record the trigger door
                # makes, with her call named as the origin.
                if _msaves:
                    try:
                        for _sv in _msaves:
                            daybook_record_capture(
                                str(_sv.get("text") or ""),
                                origin="autonomous",
                                key=_sv.get("key"),
                                proposed=bool(_sv.get("proposed")),
                                chat_id=str(data.get("chatId") or ""),
                                cfg=cfg)
                    except Exception:
                        pass
                # Her own case filings (Build 15, #161): the
                # [[finding: …]] markers come out of the reply the
                # same way, are filed into the open case in her
                # name before this payload returns, and ride back
                # as data for the dashboard's receipt.
                reply, _cfilings, _cheld = _extract_finding_markers(
                    reply, cfg, hold=_borrowed)
                # v0.5.12e (#286): no case-log parsing stands
                # between Navi and the operator — a bound-case
                # reply prints whole, exactly as composed.
                # Filing is the operator's tap on the reply
                # (case_file_reply), receipted in the room.
                # v0.5.14e (#289): the case intake retired
                # the way the case-log grammar did in v0.5.12e
                # — no parser stands between Navi and the
                # operator here either. A reply in a chat that
                # is not a case room prints whole, exactly as
                # composed; starting a case from it is the
                # operator's tap on the reply ("Start a case
                # from this" -> case_start_from_reply),
                # receipted into the chat by the bridge.
                # CLI Mode (Build 3, #209): her [[shell]] block
                # comes out of the reply the same way — the lines
                # are typed into this chat's session badged as
                # hers, and the drive rides back as data.
                # v0.4.6e (#240): the drive is gated on the turn
                # being machine-directed (shell_turn_machine_
                # directed) — computed here, where the operator's
                # words and the composer's latch both live.
                # v0.6.7e (#315) narrows the funnel the field
                # indicted: a Mesh-served reply never reaches the
                # gate at all — it is shut for borrowed text
                # outright, whatever the turn's shape — and a
                # gate that cannot be computed fails CLOSED, not
                # open: an exception is not machine intent.
                try:
                    _shell_gate = shell_turn_machine_directed(
                        last or "", data.get("chatId"), chat_mode)
                except Exception:
                    _shell_gate = False
                if _borrowed:
                    _shell_gate = False
                try:
                    reply, _shell_drove = extract_shell_block(
                        reply, data.get("chatId"),
                        gate_open=_shell_gate)
                except Exception:
                    _shell_drove = None
                # v0.4.2e (#232): the prose guard — a turn
                # whose prose came back empty (a bare shell
                # drive is the field shape) is re-asked once,
                # and never ships hollow. When the hollow turn
                # was a lender's, the words that replace it are
                # this machine's, and the record says so: the
                # mesh payload is rewritten to the local
                # fallback shape with the #224 reason word, so
                # the pill never attributes local prose to the
                # lender.
                try:
                    _pre_guard = reply
                    reply = prose_guard(reply, _shell_drove, messages,
                                        endpoint, model,
                                        chat_id=data.get("chatId"))
                    if reply != _pre_guard and _mesh_payload \
                            and _mesh_payload.get("servedBy") == "mesh":
                        _mesh_payload = {
                            "servedBy": "local", "fellBack": True,
                            "reason": "lender-error",
                            "node": _mesh_payload.get("node") or ""}
                except Exception:
                    pass
                payload = {"reply": reply, "exec": exec_block, "mode": "model", "llamafile": "running"}
                if _budget_report:
                    payload["promptBudget"] = _budget_report
                if _borrowed:
                    payload["borrowedInert"] = {
                        "memory": "proposals", "findingsHeld": _cheld,
                        "shell": "gated"}
                if _mesh_payload:
                    payload["mesh"] = _mesh_payload
                if _refpulls:
                    payload["refPulls"] = _refpulls
                if _cfilings:
                    payload["caseFilings"] = _cfilings
                if _shell_drove:
                    payload["shellDrove"] = _shell_drove
                if _msaves:
                    payload["memorySaves"] = _msaves
                if _mrefused:
                    payload["memoryRefused"] = True
                if _hits:
                    _srcs = []
                    for h in _hits:
                        _s = {"pack": h["pack"], "document": h["document"]}
                        if _s not in _srcs:
                            _srcs.append(_s)
                    payload["rag"] = _srcs
                if _vhits:
                    _all = payload.get("rag") or []
                    for h in _vhits:
                        _s = {"pack": "Vault", "document": h["path"]}
                        if _s not in _all:
                            _all.append(_s)
                    payload["rag"] = _all
                if data.get("speak"):
                    payload["spoken"] = speak_reply(reply, cfg)
                self._send(200, payload)
            except Exception as e:
                self._send(502, {"error": str(e), "exec": exec_block})
        elif self.path == "/api/chips/run":
            # Run one loaded chip by name/slug (Chips tab Run buttons,
            # same machinery chat invocations use).
            data = self._body()
            chip, cargs = match_local_chip(str(data.get("query") or ""), local_chips())
            if chip is None:
                self._send(200, {"ok": False, "error": "No chip by that name is loaded on this machine."})
            else:
                result = execute_chip(chip, cargs, cfg)
                self._send(200, {
                    "ok": True,
                    "name": chip["name"],
                    "command": result["command"],
                    "output": result["output"],
                    "exit_code": result["exit_code"],
                })
        elif self.path == "/api/vault/save":
            # Create or update a note (the editor's autosave lands
            # here). Jailed to the vault; frontmatter rides as text.
            data = self._body()
            rel = vault_join(str(data.get("path") or ""))
            if not rel or not rel.lower().endswith(".md"):
                self._send(200, {"ok": False, "error": "That path escapes the vault — notes live inside the vault folder only, as .md files."})
            else:
                vault_write(rel, str(data.get("content") or "")[:200000])
                self._send(200, {"ok": True})
        elif self.path == "/api/vault/delete":
            data = self._body()
            rel = vault_join(str(data.get("path") or ""))
            if not rel:
                self._send(200, {"ok": False, "error": "That path escapes the vault."})
            elif not os.path.isfile(vault_fs_path(rel)):
                # Never a silent no-op (build .73): if the note isn't
                # on disk, the operator hears about it in plain words.
                self._send(200, {"ok": False, "error": "That note no longer exists — it may have been moved or deleted outside the app. The tree has been refreshed."})
            else:
                vault_delete_note(rel)
                self._send(200, {"ok": True})
        elif self.path == "/api/vault/move":
            # Rename / drag-and-drop move. Never overwrites.
            data = self._body()
            src = vault_join(str(data.get("from") or ""))
            dst = vault_join(str(data.get("to") or ""))
            if not src or not dst or not dst.lower().endswith(".md"):
                self._send(200, {"ok": False, "error": "That path escapes the vault — notes live inside the vault folder only, as .md files."})
            elif os.path.exists(vault_fs_path(dst)):
                self._send(200, {"ok": False, "error": "A note already exists at " + dst + " — pick another name."})
            elif not os.path.isfile(vault_fs_path(src)):
                self._send(200, {"ok": False, "error": "That note no longer exists."})
            else:
                parent = os.path.dirname(vault_fs_path(dst))
                if parent:
                    os.makedirs(parent, exist_ok=True)
                os.replace(vault_fs_path(src), vault_fs_path(dst))
                self._send(200, {"ok": True})
        elif self.path == "/api/vault/folder":
            data = self._body()
            rel = vault_join(str(data.get("path") or ""))
            if not rel:
                self._send(200, {"ok": False, "error": "That folder path escapes the vault."})
            else:
                os.makedirs(vault_fs_path(rel), exist_ok=True)
                self._send(200, {"ok": True})
        elif self.path == "/api/vault/delete-folder":
            data = self._body()
            rel = vault_join(str(data.get("path") or ""))
            if not rel:
                self._send(200, {"ok": False, "error": "That folder path escapes the vault."})
            else:
                count = vault_delete_folder(rel)
                self._send(200, {"ok": True, "deletedNotes": count})
        elif self.path == "/api/chips/save":
            # Chip Studio saving an edit: re-forge the chip file
            # with the new code inside it.
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            name = str(data.get("name") or "").strip()[:120]
            code = str(data.get("code") or "")
            if not fname.endswith(".png") or not name or not code.strip():
                self._send(200, {"ok": False, "error": "A chip save needs a chip file, a name, and code."})
            else:
                icon_rows, icon_err = _payload_icon(data)
                if icon_err:
                    self._send(200, {"ok": False, "error": icon_err})
                    return
                payload = {
                    "name": name,
                    "description": str(data.get("description") or "")[:500],
                    "code": code[:200000],
                    "output": str(data.get("output") or "")[:5000],
                    "files": _valid_chip_files(data.get("files"), code[:200000]),
                    "glyph": str(data.get("glyph") or "")[:40],
                    "version": str(data.get("version") or "")[:40],
                    "icon": icon_rows,
                }
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, fname), "wb") as f:
                    f.write(build_cartridge_png(payload))
                # Saving an edit to a GitHub-installed chip marks the
                # divergence on its install record instead of hiding
                # it: the dashboard badges the chip as edited locally,
                # and a catalog update refuses rather than silently
                # overwriting the edits.
                rows = load_chip_installs()
                marked = False
                for r in rows:
                    if r.get("local_file") == fname and not str(r.get("full_name") or "").startswith("local/"):
                        r["locally_edited"] = True
                        marked = True
                if marked:
                    save_chip_installs(rows)
                self._send(200, {"ok": True, "filename": fname})
        elif self.path == "/api/workflows/create":
            data = self._body()
            name = str(data.get("name") or "").strip()[:120]
            steps = data.get("steps") or []
            known = {c["file"] for c in local_chips()}
            steps = [str(s) for s in steps if str(s) in known]
            if not name or len(steps) < 2:
                self._send(200, {"ok": False, "error": "A workflow needs a name and at least two chips from the folder."})
            else:
                rows = load_workflows()
                wf = {
                    "id": int(time.time() * 1000),
                    "name": name,
                    "description": str(data.get("description") or "")[:500],
                    "steps": steps,
                }
                rows.append(wf)
                save_workflows(rows)
                self._send(200, {"ok": True, "id": wf["id"]})
        elif self.path == "/api/workflows/delete":
            data = self._body()
            wid = data.get("id")
            save_workflows([w for w in load_workflows() if w.get("id") != wid])
            self._send(200, {"ok": True})
        elif self.path == "/api/workflows/run":
            data = self._body()
            wid = data.get("id")
            wf = next((w for w in load_workflows() if w.get("id") == wid), None)
            if wf is None:
                self._send(200, {"ok": False, "error": "That workflow no longer exists.", "steps": [], "finalOutput": None})
            else:
                self._send(200, run_workflow_chain(wf, cfg))
        elif self.path == "/api/rag/install":
            # Installing a pack means the pack itself lands here:
            # its documents download from the repo at the ref being
            # installed, into rag-packs/<slug>/, with the index
            # built at install time — and the record points at that
            # folder. A download that fails installs nothing.
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_rag_installs()
            if any(r.get("full_name") == full for r in rows):
                self._send(200, {"ok": False, "error": "That pack is already installed."})
            else:
                try:
                    repo = fetch_repo_chip(full)
                    documents = fetch_pack_documents(repo)
                except Exception as e:
                    self._send(200, {"ok": False, "error": "The pack could not be downloaded from GitHub, so nothing was installed: " + str(e)})
                    return
                slug = rag_pack_slug(repo, rows)
                write_rag_pack(slug, repo["name"], repo["description"] or "", documents)
                rows.append(rag_install_record(repo, slug, len(documents)))
                save_rag_installs(rows)
                self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/rag/update":
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_rag_installs()
            inst = next((r for r in rows if r.get("full_name") == full), None)
            if inst is None:
                self._send(200, {"ok": False, "updated": False, "error": "That pack is not installed."})
            elif inst.get("locally_edited"):
                self._send(200, {"ok": False, "updated": False, "error": "This pack has local edits (edited in RAG Studio), so the update was not applied — it would overwrite them. Export the edited pack for GitHub to publish your version, or delete the install and reinstall to return to the published source."})
            else:
                try:
                    repo = fetch_repo_chip(full)
                except Exception as e:
                    self._send(200, {"ok": False, "updated": False, "error": "GitHub could not be reached for the update check: " + str(e)})
                    return
                latest = repo["latestRelease"]
                if latest:
                    newer = inst.get("installed_tag") != latest["tag"]
                else:
                    newer = bool(inst.get("installed_pushed_at") and repo["pushedAt"] > inst["installed_pushed_at"])
                slug = inst.get("local_dir")
                have_pack = bool(slug) and _rag_pack_index(slug) is not None
                if not newer and have_pack:
                    self._send(200, {"ok": True, "updated": False, "error": None})
                    return
                # Newer upstream — or the record's pack folder is
                # missing its index: (re)download the documents at
                # the latest ref and rebuild. A failed download
                # leaves the installed copy exactly as it was. The
                # operator's active flag survives the update.
                try:
                    documents = fetch_pack_documents(repo)
                except Exception as e:
                    self._send(200, {"ok": False, "updated": False, "error": "The pack could not be downloaded from GitHub, so the installed copy was left as it was: " + str(e)})
                    return
                if not slug:
                    slug = rag_pack_slug(repo, rows)
                write_rag_pack(slug, repo["name"], repo["description"] or "", documents)
                fresh = rag_install_record(repo, slug, len(documents))
                fresh["installed_at"] = inst.get("installed_at")
                fresh["active"] = bool(inst.get("active", True))
                rows[rows.index(inst)] = fresh
                save_rag_installs(rows)
                self._send(200, {"ok": True, "updated": True, "error": None})
        elif self.path == "/api/rag/delete":
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_rag_installs()
            target = next((r for r in rows if r.get("full_name") == full), None)
            rows = [r for r in rows if r.get("full_name") != full]
            save_rag_installs(rows)
            # The pack's folder — documents and index — goes with
            # its record, GitHub-installed and Studio-forged alike.
            # A hand-dropped folder pack has no record; its folder
            # still goes when it is deleted by its local/ name.
            gone_dir = target.get("local_dir") if target else None
            if gone_dir is None and full.startswith("local/"):
                gone_dir = full[len("local/"):]
            if gone_dir:
                shutil.rmtree(
                    os.path.join(RAG_PACKS_DIR, gone_dir),
                    ignore_errors=True,
                )
            self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/rag/toggle":
            # Only active packs feed retrieval. The flag lives on
            # the install record; a hand-dropped folder pack gets a
            # local record the first time it is toggled.
            data = self._body()
            full = str(data.get("full_name") or "")
            active = bool(data.get("active"))
            rows = load_rag_installs()
            inst = next((r for r in rows if r.get("full_name") == full), None)
            if inst is None and full.startswith("local/"):
                slug = full[len("local/"):]
                if _rag_pack_index(slug) is not None:
                    rows.append({
                        "full_name": full,
                        "name": slug,
                        "html_url": "",
                        "installed_tag": None,
                        "installed_ref": "local pack",
                        "installed_pushed_at": None,
                        "installed_at": _now_iso(),
                        "local_dir": slug,
                        "active": active,
                        "doc_count": len((_rag_pack_index(slug) or {}).get("documents") or []),
                    })
                    save_rag_installs(rows)
                    self._send(200, {"ok": True, "error": None})
                    return
            if inst is None:
                self._send(200, {"ok": False, "error": "That pack is not installed."})
            else:
                inst["active"] = active
                save_rag_installs(rows)
                self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/rag/forge":
            # RAG Studio: package documents into a knowledge pack
            # in rag-packs/ — documents, built index, and metadata —
            # ready to activate and use immediately. The documents
            # arrive either as files the operator picked in the
            # Studio (read in the browser and sent as text; PDFs
            # travel as base64 and their text is extracted here)
            # or from a folder on this machine's disk (source_dir,
            # read here, PDFs extracted the same way).
            data = self._body()
            name = str(data.get("name") or "").strip()[:120]
            description = str(data.get("description") or "")[:500]
            documents = []
            source_dir = str(data.get("source_dir") or "").strip()
            if source_dir:
                root = os.path.abspath(os.path.expanduser(source_dir))
                if not os.path.isdir(root):
                    self._send(200, {"ok": False, "error": "That folder is not on this machine: " + source_dir})
                    return
                found = []
                for dirpath, _dirs, files in os.walk(root):
                    for fn in sorted(files):
                        if fn.lower().endswith((".md", ".txt", ".pdf")):
                            found.append(os.path.join(dirpath, fn))
                found.sort()
                if not found:
                    self._send(200, {"ok": False, "error": "No .md, .txt or .pdf documents in that folder — a pack is made of Markdown, text, or PDF documents."})
                    return
                if len(found) > RAG_MAX_DOCS:
                    self._send(200, {"ok": False, "error": "That folder holds " + str(len(found)) + " documents — a pack caps at " + str(RAG_MAX_DOCS) + "."})
                    return
                for path in found:
                    if path.lower().endswith(".pdf"):
                        try:
                            with open(path, "rb") as f:
                                body = pdf_extract_text(f.read())
                        except Exception as e:
                            self._send(200, {"ok": False, "error": "Could not read " + path + ": " + str(e)})
                            return
                        if not body: