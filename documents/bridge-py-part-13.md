# bridge.py — part 13 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 46082–46748 of bridge.py.
Top-level symbols in this part: probe_bridge, probe_port_holder, pid_alive, pid_cmdline, port_holder_pid, claim_pid_file, drop_pid_file, watchdog, run_self_check.


def probe_bridge(port):
    # What is already answering on the bridge port, if anything:
    # the /api/status JSON, or None when nothing (or something that
    # is not a Navi bridge) answers.
    try:
        with urllib.request.urlopen(
            "http://127.0.0.1:" + str(port) + "/api/status", timeout=2
        ) as resp:
            data = json.loads(resp.read().decode())
        return data if isinstance(data, dict) else None
    except Exception:
        return None


def probe_port_holder(port):
    # Identify whatever is serving the bridge port when it is NOT a
    # Navi bridge: fetch its front page and lift the <title> out of
    # it, so messages can name the incumbent — the operator's own
    # earlier UI for Navi also served this port, and "something else
    # is on 8675" is a useless diagnosis next to its page's own name.
    # Returns {"navi": True} for a Navi bridge of any build, a dict
    # describing the foreign page, or None when nothing answers.
    status = probe_bridge(port)
    if status is not None and status.get("app") == "Navi: Apex":
        return {"navi": True, "build": status.get("bridge")}
    try:
        req = urllib.request.Request(
            "http://127.0.0.1:" + str(port) + "/",
            headers={"User-Agent": "navi-bridge-probe"},
        )
        with urllib.request.urlopen(req, timeout=2) as resp:
            raw = resp.read(65536).decode("utf-8", "replace")
    except Exception:
        return None
    m = re.search(r"<title[^>]*>(.*?)</title>", raw, re.I | re.S)
    title = re.sub(r"\s+", " ", m.group(1)).strip() if m else ""
    return {"navi": False, "title": title or "(a page with no title)"}


def pid_alive(pid):
    try:
        os.kill(pid, 0)
        return True
    except Exception:
        return False


def pid_cmdline(pid):
    # The process's command line as one line of text, read from
    # /proc — used to NAME a port holder in startup messages
    # (build .91), never to decide anything about it.
    try:
        with open("/proc/" + str(pid) + "/cmdline", "rb") as f:
            raw = f.read()
        return " ".join(part.decode("utf-8", "replace") for part in raw.split(bytes([0]))).strip()
    except Exception:
        return ""


def port_holder_pid(port):
    # The PID holding the LISTEN socket on the port, resolved from
    # /proc alone (build .91, issue #91): names a holder that
    # answers NOTHING — a bridge frozen or stuck so the status
    # probe and the page-title probe both come back empty, the
    # state a wedged update leaves behind once the pid file is
    # gone. Returns None when the port is free or the holder
    # cannot be resolved.
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


def claim_pid_file():
    try:
        with open(PID_PATH, "w") as f:
            f.write(str(os.getpid()))
    except Exception:
        pass


def drop_pid_file():
    try:
        with open(PID_PATH) as f:
            mine = f.read().strip() == str(os.getpid())
        if mine:
            os.remove(PID_PATH)
    except Exception:
        pass


def watchdog(started_mtime):
    # Exit with code 75 ("replaced — relaunch me") when this process
    # has been superseded: bridge.py changed on disk (a reinstall
    # landed while we were running), or a newer bridge claimed the
    # pid file and is waiting for this port. The launchers relaunch
    # the bridge on 75, so an update takes over without the operator
    # hunting down a zombie process.
    self_path = os.path.abspath(__file__)
    while True:
        time.sleep(2)
        try:
            if started_mtime and os.path.getmtime(self_path) != started_mtime:
                os._exit(75)
        except SystemExit:
            raise
        except Exception:
            pass
        try:
            with open(PID_PATH) as f:
                owner = int((f.read() or "").strip() or 0)
            if owner and owner != os.getpid() and pid_alive(owner):
                os._exit(75)
        except SystemExit:
            raise
        except Exception:
            pass


def run_self_check():
    # 0.2.0e Build 10 (#193, installer rebuild piece C): the
    # product's own verdict on the install it is running from —
    # never a version proxy. Three proofs, all against the live
    # tree beside this file: (1) the stamps agree — the SHIPPED_VERSION
    # file, the config's stamped version, and the stamps baked
    # into this bridge are one build; (2) every payload file
    # the package's MANIFEST.txt declares verifies by sha256
    # against the bytes in place; (3) every runtime engine
    # imports from THIS directory. The verdict lands in
    # config/install-self-check.json and prints as one named
    # sentence: SELF-CHECK OK / SELF-CHECK FAILED. Exit 0 only
    # when all three proofs hold.
    root = ROOT
    problems = []
    checked = 0
    disk_version = ""
    try:
        with open(os.path.join(root, "VERSION"), encoding="utf-8") as f:
            disk_version = f.read().strip()
    except Exception:
        problems.append("the SHIPPED_VERSION file is missing from this install")
    try:
        cfg = load_config()
    except Exception:
        cfg = {}
    # The witnesses, the same ones the updater reads: the
    # SHIPPED_VERSION file (number + status word), the config's stamped
    # version/version_status, and the stamps baked into this
    # bridge. All must name the same build.
    if disk_version:
        parts = disk_version.splitlines()[0].split() if disk_version.splitlines() else []
        if not parts or parts[0] != SHIPPED_VERSION:
            problems.append("the SHIPPED_VERSION file reads " + disk_version
                            + " but this bridge is " + SHIPPED_VERSION)
        elif len(parts) > 1 and parts[1] != SHIPPED_VERSION_STATUS:
            problems.append("the SHIPPED_VERSION file's status word is " + parts[1]
                            + " but this bridge is " + SHIPPED_VERSION_STATUS)
    if cfg.get("version") and cfg.get("version") != SHIPPED_VERSION:
        problems.append("the config's stamped version is " + str(cfg.get("version"))
                        + " but this bridge is " + SHIPPED_VERSION)
    if cfg.get("version_status") and cfg.get("version_status") != SHIPPED_VERSION_STATUS:
        problems.append("the config's stamped version status is "
                        + str(cfg.get("version_status")) + " but this bridge is "
                        + SHIPPED_VERSION_STATUS)
    man_path = os.path.join(root, "MANIFEST.txt")
    entries = []
    if os.path.isfile(man_path):
        try:
            with open(man_path, encoding="utf-8") as f:
                for line in f:
                    line = line.strip()
                    if not line or line.startswith("#"):
                        continue
                    parts = line.split("\t")
                    if len(parts) == 3 and parts[1] == "payload":
                        entries.append((parts[0], parts[2]))
        except Exception:
            problems.append("MANIFEST.txt cannot be read in this install")
    else:
        problems.append("MANIFEST.txt is missing from this install — the payload cannot be verified")
    for rel, want in entries:
        if want == "-":
            continue
        p = os.path.join(root, rel)
        if not os.path.isfile(p):
            problems.append(rel + " is missing from this install")
            continue
        h = hashlib.sha256()
        try:
            with open(p, "rb") as f:
                for chunk in iter(lambda: f.read(65536), b""):
                    h.update(chunk)
        except Exception:
            problems.append(rel + " cannot be read in this install")
            continue
        if h.hexdigest() != want:
            problems.append(rel + " does not match the digest its manifest declares")
        else:
            checked += 1
    if os.path.isfile(man_path) and not entries:
        problems.append("MANIFEST.txt declares no payload files")
    for mod in ("guardian", "intel", "wire", "ctf", "bounties"):
        try:
            spec = importlib.util.spec_from_file_location(
                "selfcheck_" + mod, os.path.join(root, mod + ".py"))
            m = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(m)
        except Exception as e:
            problems.append(mod + ".py does not import from this install ("
                            + type(e).__name__ + ")")
    # 0.2.0e Build 11 (#198, the permissions doctrine): the
    # ownership verdict over this tree — every file should be
    # owned by the operator running the bridge and readable by
    # them. A root-owned leftover (a past sudo repair's copies,
    # a voice venv installed under sudo) is exactly how an
    # install quietly rots until an update dies on it; the
    # verdict names the offenders in the record instead. It
    # does NOT fail the proofs above — the install stands on
    # those — it is the product saying the rest out loud.
    # Symlinks are skipped (their targets live outside the
    # tree's ownership story), and so is updates/ (staged
    # downloads, not the install itself).
    ownership_offenders = []
    ownership_count = 0
    try:
        _uid = os.getuid()

        def _own_bad(p, is_dir):
            try:
                st = os.lstat(p)
            except Exception:
                return False
            if st.st_uid != _uid:
                return True
            if not os.access(p, os.R_OK):
                return True
            if is_dir and not os.access(p, os.X_OK):
                return True
            return False

        for _dp, _dns, _fns in os.walk(root):
            _rel_dp = os.path.relpath(_dp, root)
            _kept = []
            for _d in _dns:
                _p = os.path.join(_dp, _d)
                if os.path.islink(_p):
                    continue
                if _rel_dp == "." and _d == "updates":
                    continue
                _kept.append(_d)
                if _own_bad(_p, True):
                    ownership_count += 1
                    if len(ownership_offenders) < 50:
                        ownership_offenders.append(
                            os.path.relpath(_p, root))
            _dns[:] = _kept
            for _n in _fns:
                _p = os.path.join(_dp, _n)
                if os.path.islink(_p):
                    continue
                if _own_bad(_p, False):
                    ownership_count += 1
                    if len(ownership_offenders) < 50:
                        ownership_offenders.append(
                            os.path.relpath(_p, root))
    except Exception:
        ownership_offenders = []
        ownership_count = 0
    ownership_ok = ownership_count == 0
    ok = not problems
    if ok:
        body = ("the stamps agree (build " + BRIDGE_BUILD
                + ", sequence " + str(SHIPPED_BUILD_SEQ) + "), all " + str(checked)
                + " manifest payload files verify byte-for-byte, and every "
                "runtime engine imports from this install.")
        sentence = "Self-check passed: " + body
    else:
        body = "; ".join(problems) + "."
        sentence = "Self-check FAILED: " + body
    rec = {"ok": ok, "sentence": sentence, "build": BRIDGE_BUILD,
           "buildSeq": SHIPPED_BUILD_SEQ, "version": SHIPPED_VERSION,
           "payloadVerified": checked, "problems": problems,
           "ownershipOk": ownership_ok,
           "ownershipOffenders": ownership_offenders,
           "ownershipOffenderCount": ownership_count,
           "at": int(time.time())}
    try:
        with open(os.path.join(root, "config", "install-self-check.json"), "w", encoding="utf-8") as f:
            json.dump(rec, f, indent=1)
    except Exception:
        pass
    print(("SELF-CHECK OK: " if ok else "SELF-CHECK FAILED: ") + body, flush=True)
    if ownership_ok:
        print("SELF-CHECK OWNERSHIP: ok — every file in this tree is owned by "
              "the operator running the bridge and readable by them.", flush=True)
    else:
        print("SELF-CHECK OWNERSHIP: " + str(ownership_count)
              + " file(s) in this tree are not owned by the operator running "
              "the bridge, or cannot be read by them: "
              + ", ".join(ownership_offenders)
              + ("…" if ownership_count > len(ownership_offenders) else "")
              + " — the install stands, but repair ownership (sudo chown -R) "
              "before the next update has to carry them.", flush=True)
    return 0 if ok else 1


if __name__ == "__main__":
    # 0.2.0e Build 7: the access code's deliberate doors, for a
    # headless or container install where no dashboard is open
    # yet — the recovery one-liners the Docker docs carry:
    #   python3 bridge.py --dhd-code         print the current
    #                                        code (minting one
    #                                        first if none was
    #                                        ever set)
    #   python3 bridge.py --dhd-code-reset   mint a NEW code and
    #                                        print it; the old
    #                                        code stops knocking
    #                                        at once (devices
    #                                        already trusted keep
    #                                        their own tokens,
    #                                        exactly as the
    #                                        dashboard's
    #                                        regenerate behaves)
    # Both end here, before any server starts; ordinary boots
    # never print the code anywhere.
    if len(sys.argv) > 1 and sys.argv[1] in ("--dhd-code", "--dhd-code-reset"):
        _cfg = load_config()
        if sys.argv[1] == "--dhd-code-reset":
            _block = _cfg.get("public_server") if isinstance(_cfg.get("public_server"), dict) else {}
            _block = dict(_block)
            _block["access_code"] = generate_access_code()
            if "enabled" not in _block:
                _block["enabled"] = False
            _cfg["public_server"] = _block
            save_config(_cfg)
            print(_block["access_code"])
        else:
            print(ensure_access_code(_cfg))
        sys.exit(0)
    # 0.2.0e Build 10 (#193): the install self-check — the
    # installer runs this against the freshly promoted tree
    # before it may call the install done, and an operator can
    # run it any time: python3 bridge.py --self-check. It ends
    # here, before any server starts.
    if len(sys.argv) > 1 and sys.argv[1] == "--self-check":
        sys.exit(run_self_check())
    cfg = load_config()
    # Automatic profiling (Build 14, #157): a config with no
    # hardware profile profiles this machine once, here, before
    # anything reads the runtime settings — guarded inside, so a
    # failure leaves the fluid-floor behavior exactly as before.
    cfg = auto_profile_on_startup(cfg)
    # The port is this build's own constant (BRIDGE_PORT), never the
    # config's: a navi.config.json carried across from an older
    # package build records that build's port, and honoring it once
    # bound this bridge to the old address while its launcher polled
    # the new one — the bridge cheerfully reported itself "already
    # running" on a port nothing was waiting on. A disagreeing config
    # value is named, then ignored.
    port = BRIDGE_PORT
    try:
        cfg_port = int(cfg.get("bridge", {}).get("port") or 0)
    except Exception:
        cfg_port = 0
    if cfg_port and cfg_port != port:
        print("Note: config/navi.config.json names bridge port "
              + str(cfg_port) + ", left over from an older install — "
              "ignoring it; this build's bridge port is "
              + str(port) + ".", flush=True)
    existing = probe_bridge(port)
    if existing and existing.get("app") == "Navi: Apex":
        if existing.get("bridge") == BRIDGE_BUILD:
            print("Navi bridge (build " + BRIDGE_BUILD + ") is already "
                  "running on http://localhost:" + str(port)
                  + " — the dashboard is live at that address; "
                  "nothing to do.", flush=True)
            sys.exit(0)
        # An older build holds the port. Claim the pid file: a bridge
        # new enough to watch it exits and frees the port; the bind
        # loop below waits for that (and reports plainly if the old
        # process predates the watchdog and has to be quit by hand).
        print("An older Navi bridge (build "
              + str(existing.get("bridge") or "pre-stamp") + ") is answering "
              "on port " + str(port) + " — taking over for build "
              + BRIDGE_BUILD + ".", flush=True)
    else:
        # Not a Navi bridge. If a different program is serving the
        # port — the operator's own earlier UI for Navi also lived at
        # this address — say so now, by name: the bind loop below can
        # only fail against an incumbent that will never step aside,
        # and "an older bridge holds the port" would send the operator
        # hunting for a process that doesn't exist.
        holder = probe_port_holder(port)
        if holder and not holder.get("navi"):
            # A foreign program will never step aside for the pid
            # watchdog, so there is nothing to wait for: name it and
            # stop. (The bind loop keeps the same diagnosis as a
            # backstop for a holder that appears after this probe.)
            print("Port " + str(port) + " is already served by a "
                  "different program — its page is titled \""
                  + str(holder.get("title")) + "\". That program, not "
                  "this bridge, is what a browser shows at "
                  "http://localhost:" + str(port) + " until it stops. "
                  "Quit it (find it with: lsof -i :" + str(port)
                  + ") and launch Apex again.", flush=True)
            sys.exit(1)
        if holder is None:
            # Nothing answers on the port — but something may still
            # HOLD it: a bridge frozen or stuck so it accepts no
            # requests at all (build .91, the wedged-update state,
            # its pid file long gone). Resolve the holder from the
            # socket itself and name it, so the bind loop's wait —
            # and its failure message, if it comes to that — is
            # about a named process, not a shrug.
            _hp = port_holder_pid(port)
            if _hp:
                print("Port " + str(port) + " is held by process "
                      + str(_hp) + " ("
                      + (pid_cmdline(_hp) or "unknown command")
                      + "), which is not answering — waiting for it "
                      "to step aside.", flush=True)
    claim_pid_file()
    atexit.register(drop_pid_file)
    # CLI Mode shell sessions are children of this process: close
    # them on the way out so no interactive shell outlives the
    # bridge. Their masters are pipes owned by this process, so
    # even a hard kill leaves them reading EOF — this is the tidy
    # path, not the only one.
    atexit.register(shell_sessions_kill_all)
    try:
        _mtime = os.path.getmtime(os.path.abspath(__file__))
    except Exception:
        _mtime = 0
    threading.Thread(target=watchdog, args=(_mtime,), daemon=True).start()
    # GitHub interval backups: checks on launch and while the bridge
    # runs whether a backup is due on the operator's chosen interval,
    # and runs it when signed in. Daemon thread; dies with the bridge.
    threading.Thread(target=github_auto_backup_loop, daemon=True).start()
    # Build 3 (#176): Quick Sync's app-start pull — the third
    # trigger beside the Sync control and DHD session start. It
    # only pulls and converges; publishing stays a deliberate act.
    threading.Thread(target=sync_startup_pull, daemon=True).start()
    # Scheduled warm (Build 14, #157): checks runtime.warm_at once
    # a minute and, at the configured minute, starts + warms a
    # llamafile that is not answering. Daemon thread; dies with
    # the bridge; inert while warm_at is unset.
    threading.Thread(target=_warm_at_loop, daemon=True).start()
    # Guardian (Build 14, #156): the headroom sampler starts with
    # the bridge's other background services whenever its saved
    # setting is on (headless included — this block runs before
    # the bind either way). The engine re-checks enabled on every
    # tick regardless, so a settings save can quiet it live.
    try:
        if guardian_settings(cfg)["enabled"]:
            GUARDIAN.start()
            print("Guardian is ON: watching headroom (reserve "
                  + str(guardian_settings(cfg)["reserveMb"]) + " MB).", flush=True)
    except Exception:
        pass
    # Threat Intelligence Center (Build 15, #165): the
    # collection scheduler starts with the bridge's other
    # background services. The engine re-checks its enabled
    # setting every cycle, and a node named as a receiver pulls
    # its collector's digest over the Mesh door instead of
    # polling the feeds itself.
    try:
        WIRE.start()
        CTF.start()
        BOUNTIES.start()
        pulse_start()
        seraph_start()
        daybook_start()
    except Exception:
        pass
    try:
        if intel_settings(cfg)["enabled"]:
            INTEL.start()
            print("Threat Intelligence Center is ON: "
                  + ("receiving from a Mesh collector." if intel_settings(cfg)["collectorPeer"]
                     else "collecting on the scheduled cadence."), flush=True)
    except Exception:
        pass
    # Local server (build .59): the saved choice picks the bind
    # for THIS process, once, here. ON binds every interface so
    # the LAN can reach the port; OFF keeps the loopback-only
    # bind Apex has always had. Internal callers poll 127.0.0.1,
    # which answers on either bind, so nothing else changes.
    if dhd_settings(cfg)["enabled"]:
        # DHD mode (build .74): the bridge NEVER binds publicly -
        # Apex's own proxy is the only public face, forwarding to
        # this loopback bind. This wins over the Local server
        # choice, which is about the LAN door on the bridge port.
        ensure_access_code(cfg)
        ACTIVE_BIND_HOST = "127.0.0.1"
    elif public_settings(cfg)["enabled"]:
        ensure_access_code(cfg)
        ACTIVE_BIND_HOST = "0.0.0.0"
    else:
        ACTIVE_BIND_HOST = "127.0.0.1"
    # Local server restart reconciliation (build .61): if the
    # previous bridge recorded a restart request before exiting,
    # this bridge is its answer — verify this process's actual bind
    # against the saved choice and record the verdict where the
    # Settings row reads it. A match confirms; a mismatch fails at
    # the verify step and names the state that is actually live.
    _ps = public_settings(cfg)
    if _ps["restart_status"] == "restarting":
        if _ps["enabled"] == (ACTIVE_BIND_HOST == "0.0.0.0"):
            set_public_restart(cfg, "confirmed", "", "Restart complete — the bridge that answers now reports " + ("Local — on your network." if _ps["enabled"] else "This machine only."))
        else:
            set_public_restart(cfg, "failed", "verify", "A bridge answered after the restart, but its bind does not match the saved choice, so the change is not confirmed. Still live: " + ("Local — on your network (the old setting)." if ACTIVE_BIND_HOST == "0.0.0.0" else "This machine only — 127.0.0.1 (the old setting).") + " Press Restart bridge in Settings to try again, or quit Apex fully and relaunch.")
    # Threading: a connect that starts the server can wait minutes
    # for the model to load; it must not block status and chat.
    server = None
    deadline = time.time() + 30
    while server is None:
        try:
            server = ThreadingHTTPServer((ACTIVE_BIND_HOST, port), Handler)
        except OSError:
            if time.time() >= deadline:
                # Name the actual incumbent — an older Navi bridge
                # that never stepped aside, or a different program
                # entirely — instead of always blaming the bridge.
                holder = probe_port_holder(port)
                if holder and not holder.get("navi"):
                    print("Port " + str(port) + " is held by a different "
                          "program (its page is titled \""
                          + str(holder.get("title")) + "\"), not by a "
                          "Navi bridge — it will never step aside for "
                          "this one. Quit that program (find it with: "
                          "lsof -i :" + str(port) + ") and launch "
                          "Apex again.", flush=True)
                else:
                    # Name the holder by PID when the socket itself
                    # can resolve it (build .91): "end the python
                    # bridge.py process" sent the operator hunting
                    # through a process list for a process nothing
                    # would identify — the exact state a wedged
                    # update leaves behind.
                    _hp = port_holder_pid(port)
                    if _hp:
                        print("Port " + str(port) + " is still held by process "
                              + str(_hp) + " ("
                              + (pid_cmdline(_hp) or "unknown command")
                              + "), and it did not step aside. End it with: "
                              "kill -9 " + str(_hp)
                              + " — then launch Apex again.", flush=True)
                    else:
                        print("Port " + str(port) + " is still held by the older "
                              "bridge, and it did not step aside — it predates the "
                              "self-update watchdog. Quit the running Navi bridge "
                              "(close the app, or end the python bridge.py process) "
                              "and launch again.", flush=True)
                sys.exit(1)
            time.sleep(1)
    print("Navi bridge (build " + BRIDGE_BUILD + ") on http://localhost:" + str(port), flush=True)
    # Background update re-check (build .86): the bridge keeps its
    # own update verdict fresh while it runs (see _update_bg_loop).
    threading.Thread(target=_update_bg_loop, daemon=True).start()
    if ACTIVE_BIND_HOST == "0.0.0.0":
        print("Local server is ON: the dashboard also answers on the local network at "
              + public_state_payload(load_config(), True)["lanUrl"]
              + " - visitors from other machines need the access code from Settings.", flush=True)
    # DHD (build .74): if the saved choice is on, bring the public
    # door up now - certificate, proxy, self-check - and start the
    # daily renewal watch. A failed bring-up names itself in the
    # DHD state the dashboard reads; the bridge itself serves on.
    if dhd_settings(cfg)["enabled"]:
        threading.Thread(target=dhd_renewal_loop, daemon=True).start()
        if dhd_bring_up(load_config()):
            _dp = dhd_state_payload(load_config(), True)
            print("DHD is ON: " + _dhd_on_summary(_dp)
                  + " The proxy on 80/443 is the only public face; visitors need the access code.", flush=True)
        else:
            print("DHD is saved ON but did not come up: " + DHD_STATE["last_error"], flush=True)
        # If an enable operation was waiting on this restart, its
        # persisted card now reconciles with the live state.
        dhd_resume_enable_operation()
    # Federation (Build 8): a gateway card waiting at
    # config/federation/invitation.png — placed there by a
    # container image's first boot, or by an operator's hand — is
    # imported admission-only, once, before the doors open.
    try:
        _fed_invited = federation_import_invitation_file()
        if _fed_invited:
            print("Federation: admitted to '" + _fed_invited + "' by its "
                  "waiting gateway card (admission + introduction only).", flush=True)
    except Exception:
        pass
    # Case Files (v0.5.3e, #265/#266): the migration sweep —
    # every case folder on the desk is cleaned of machinery
    # text earlier builds wrote into its findings, and its one
    # full case file is formed in the Vault, before the doors
    # open. Idempotent; a desk already formed is untouched.
    try:
        _formed = case_form_all()
        if _formed:
            print("Case Files: formed " + str(len(_formed))
                  + " full case file(s) in the Vault.", flush=True)
    except Exception:
        pass
    # Dossiers shelf (v0.6.4e, #320): every dossier stands as
    # one file under Dossiers/ before the doors open, and a
    # dossier no longer in the store leaves no file behind.
    try:
        _dformed = dossier_form_all()
        if _dformed:
            print("Dossiers: formed " + str(len(_dformed))
                  + " dossier file(s) on the Vault shelf.", flush=True)
    except Exception:
        pass
    # Sync account gate (v0.6.4e, #323): the first gated run
    # ledgers the standing corpus's attribution and, when this
    # seat is signed in, judges it — material that cannot be
    # attributed to this seat's account is purged before the
    # doors open, with its receipt standing in the sync state.
    try:
        _purge = sync_account_gate_sweep()
        if _purge and _purge.get("fact"):
            print("Sync: " + _purge["fact"], flush=True)
    except Exception:
        pass
    # Navi Mesh (Ring 1; dedicated door since build .86): if Mesh
    # is on, open its dedicated TLS door on its listen ports —
    # always, whether or not DHD is on. Mesh never rides the DHD
    # door; the proxy refuses /mesh/* outright.
    if mesh_settings(cfg)["enabled"]:
        mesh_start_listener()
        if MESH_SERVERS:
            print("Navi Mesh is ON: the dedicated Mesh door listens on ports "
                  + ", ".join(str(p) for p in mesh_settings(cfg)["listenPorts"]) + " (TLS, fingerprint-pinned).", flush=True)
        else:
            print("Navi Mesh is ON but its dedicated door did not open — see the Mesh page.", flush=True)
    server.serve_forever()
