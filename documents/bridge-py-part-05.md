# bridge.py — part 5 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 16275–20497 of bridge.py.
Top-level symbols in this part: main, _guarded_main, _apply_helper_spawn_argv, _run_update_apply, update_apply_payload, save_download_payload, github_signin_client_id, github_read_token, github_api, _github_api_once, github_api_raw, _github_api_raw_once, github_err_text, github_settings, save_github_settings, github_auth, github_save_auth, github_refresh_auth, github_ensure_fresh_token, github_status_payload, github_issues_payload, github_pulse_payload, github_device_start, github_device_poll, _dashboard_image_field, _dashboard_image_bytes, github_collect_backup, github_run_backup, github_fetch_backup, github_import_changes, _github_safe_part, github_apply_import, github_backup_due, github_auto_backup_loop, load_chip_installs, save_chip_installs, fetch_repo_chip, chip_install_record, chip_slug, chip_cartridge_filename, github_download, _github_download_once, fetch_chip_payload, _display_ref, chip_catalog_view, _png_chunk, valid_icon_art, _icon_art_of, _manifest_icon, _payload_icon, _tile_field_seed, _draw_tile_field, _draw_spray, _draw_silhouette, pick_chip_glyph, _cartridge_face_png, build_cartridge_png, _valid_chip_files, parse_cartridge_png, deck_contents, _png_text_keywords, local_chips, match_local_chip, _chip_keys, resolve_chip_invocation, chip_shelf_reply, chip_invocation, _NaviInstance, get_parameters.


def main():
    job_path = sys.argv[1] if len(sys.argv) > 1 else ""
    if job_path:
        STATE["trace_path"] = os.path.join(
            os.path.dirname(os.path.abspath(job_path)),
            "apply-trace.log")
    trace("helper-start",
          "pid " + str(os.getpid()) + ", job " + (job_path or "(none)"))
    for sig in (signal.SIGTERM, signal.SIGINT, signal.SIGHUP):
        try:
            signal.signal(sig, _on_signal)
        except Exception as e:
            trace("signal-handler-skipped", str(e))
    beat = threading.Thread(target=_heartbeat, daemon=True)
    beat.start()
    with open(job_path) as f:
        job = json.load(f)
    STATE["job"] = job
    if job.get("traceLog"):
        STATE["trace_path"] = job["traceLog"]
    root = job["root"]
    STATE["result_path"] = job["resultPath"]
    pkg_dir = job["pkgDir"]
    trace("job-loaded",
          "target " + str(job.get("targetLabel"))
          + ", target build " + str(job.get("targetBuildSeq")))
    # v0.4.3e (#237): before the installer runs, the staged
    # package is copied to the holding folder beside the
    # install root (see _stage_holding_dir) — the one copy of
    # the package the installer's swap and its end-of-run
    # cleanup can never take, so _recover_stage_artifacts
    # can always put the manual fallback's folder back where
    # the fallback command points.
    try:
        _held = _stage_holding_dir(job)
        if _held and os.path.isdir(pkg_dir):
            _held_pkg = os.path.join(_held, "package")
            if os.path.isdir(_held_pkg):
                shutil.rmtree(_held_pkg, ignore_errors=True)
            os.makedirs(_held, exist_ok=True)
            shutil.copytree(pkg_dir, _held_pkg)
            trace("stage-held",
                  "the staged package is held at " + _held_pkg
                  + " while the installer runs")
    except Exception as e:
        trace("stage-hold-failed", str(e))

    # v0.3.0e Build 1 (#205): takeover mode. When the staged
    # package's bootstrap spawned this helper, it takes an apply
    # over from the helper the RUNNING (older) build started:
    # the installer that old helper ran handed off and exited
    # instead of installing. Arm first — the marker file is what
    # the handing-off installer waits for — then wait for the
    # old helper to finish its own bookkeeping and die. Two
    # helpers never work the same apply at once, and this one
    # signals nobody: it only ever waits. (The invariant is
    # untouched: no step of an apply signals any process in its
    # own ancestry — a predecessor is not an ancestor.)
    _wait_pid = job.get("takeoverWaitPid")
    if _wait_pid:
        trace("takeover-armed",
              "the incoming build's helper takes this apply over "
              "from helper pid " + str(_wait_pid)
              + " — the old helper is never signalled; it is "
              "waited out")
        _marker = job.get("takeoverMarker")
        if _marker:
            try:
                with open(_marker, "w") as _mf:
                    _mf.write(str(os.getpid()))
            except Exception:
                pass
        _t0 = time.time()
        while not _pid_gone(int(_wait_pid)) and time.time() - _t0 < 180:
            time.sleep(0.5)
        trace("takeover-proceed",
              "the previous helper is gone (waited "
              + str(int(time.time() - _t0)) + "s); the apply "
              "continues under the incoming build's machinery")

    env = dict(os.environ)
    env["NAVI_HOME"] = root
    if job.get("bootstrap"):
        # The installer checks this before its own handoff: an
        # apply the package's machinery already governs must not
        # hand off a second time.
        env["APEX_APPLY_BOOTSTRAPPED"] = "1"
    STATE["env"] = env

    def relaunch():
        try:
            if _relaunch_systemd_owned(job):
                # v0.6.7e (#305): systemd owns the restart on
                # a boot-startup seat — signal the unit and
                # let the confirmation loop below prove the
                # new build answers. No start.sh is spawned.
                try:
                    subprocess.run(
                        ["systemctl", "--user", "restart",
                         "navi-bridge.service"],
                        stdin=subprocess.DEVNULL,
                        stdout=subprocess.DEVNULL,
                        stderr=subprocess.DEVNULL, timeout=60)
                    trace("relaunch-systemd",
                          "systemd owns this install's bridge "
                          "(--startup boot) — navi-bridge.service "
                          "was signalled to restart; no start.sh "
                          "was spawned")
                except Exception as e:
                    trace("relaunch-systemd",
                          "systemd owns this install's bridge "
                          "(--startup boot); the restart signal "
                          "could not be sent (" + str(e) + ") — "
                          "the unit's own state stands, and the "
                          "confirmation below judges what answers")
                return ""
            # The relaunch log lives in the stage folder, which
            # the installer's swap parks with the previous tree
            # on a same-home apply: without its folder recreated
            # this open fails, the relaunch never starts, and a
            # landed update records itself unconfirmed (the
            # trace goes silent the same way — trace() recreates
            # its own folder for the same reason).
            _rdir = os.path.dirname(job["relaunchLog"])
            if _rdir:
                os.makedirs(_rdir, exist_ok=True)
            log = open(job["relaunchLog"], "ab")
            # v0.4.6e (#185): a headless install relaunches
            # headless — start.sh --headless starts the services
            # and opens no window (and attempts no browser
            # fallback on a seat with no browser session). The
            # determination is read at call time: after the
            # installer ran, the installed config carries this
            # install's own recorded verdict.
            _rl_argv = ["bash", os.path.join(root, "start.sh")]
            try:
                _rl_headless, _rl_why = _install_headless(job, root)
            except Exception:
                _rl_headless, _rl_why = False, ""
            if _rl_headless:
                _rl_argv.append("--headless")
                trace("relaunch-mode",
                      "headless (#185): " + _rl_why)
            subprocess.Popen(
                _rl_argv,
                cwd=root, env=env, stdin=subprocess.DEVNULL,
                stdout=log, stderr=subprocess.STDOUT,
                start_new_session=True, close_fds=True)
            trace("relaunch", "start.sh spawned")
            return ""
        except Exception as e:
            trace("relaunch-failed", str(e))
            return str(e)

    # Give the bridge a moment to answer the apply request and
    # the panel a moment to paint the installing step before the
    # installer stops the bridge.
    time.sleep(2)
    STATE["step"] = "installing"
    write_result({"state": "working", "step": "installing"})
    startup = "boot" if job.get("startup") == "on-boot" else "launch"
    # The install's known answers ride as flags (build .84, issue
    # #75): this run is detached and terminal-less by design, so
    # the installer is never left to ask a question nobody can
    # answer — startup from the config (above), voice from whether
    # her venv is on disk, both recorded on the job by the bridge
    # when it staged the update.
    voice = "yes" if job.get("voice") == "yes" else "no"
    # v0.4.6e (#185): the headless determination rides the
    # installer invocation, so the installer's own record of
    # this apply carries it. The installer ALSO decides for
    # itself (flag / boot implication / recorded / auto-detect,
    # in that order), so a helper that cannot see the whole
    # picture never strands the determination — but when this
    # helper CAN see it (above), it says it out loud, as a flag.
    _hl_pre, _hl_pre_source = _install_headless(job, root)
    code = None
    why = ""
    trace("installer-start",
          "bash install.sh --startup " + startup + " --voice " + voice
          + (" --headless" if _hl_pre else "")
          + " (headless: " + ("yes — " if _hl_pre else "no — ")
          + _hl_pre_source + ")")
    # v0.4.3e (#237): the install log's handle stays OPEN
    # after the installer exits — it is the rescue's source
    # (see _recover_stage_artifacts): the swap renames the
    # tree the log file sits in, and a completed installer
    # deletes that tree, but an open handle reads the content
    # back through both.
    _log_handle = None
    try:
        # "a+b", not a write-only mode: the recovery below
        # reads the log back through this same handle after
        # the swap, and a write-only handle cannot be read.
        _log_handle = open(job["installLog"], "a+b")
        _install_argv = ["bash", os.path.join(pkg_dir, "install.sh"),
                         "--startup", startup, "--voice", voice,
                         "--dir", root]
        if _hl_pre:
            # #185: the determination, carried as the flag the
            # installer records (see the trace line above).
            _install_argv.append("--headless")
        proc = subprocess.run(
            _install_argv,
            cwd=pkg_dir, env=env, stdout=_log_handle,
            stderr=subprocess.STDOUT, timeout=3000)
        code = proc.returncode
    except subprocess.TimeoutExpired:
        why = "the installer did not finish within 50 minutes"
    except Exception as e:
        why = str(e)
    trace("installer-exit",
          "code " + str(code) + ((", " + why) if why else ""))
    # v0.3.0e Build 2 (#208): the bootstrap handoff's verdict,
    # quoted into the trace. The Build 1 installer gate decided
    # in silence — a declined handoff left no named trace in
    # the record, the install log, or here, so the field could
    # not say afterwards WHY the incoming build's machinery did
    # not govern its own pull. The installer now writes its
    # verdict (go / no / refused, with the reason) beside the
    # job; the helper quotes it into the step log, the diary
    # the operator actually reads.
    try:
        _verdict_path = os.path.join(
            os.path.dirname(os.path.abspath(job_path)),
            "apply-handoff-verdict.json")
        with open(_verdict_path) as _vf:
            _verdict = json.load(_vf)
        trace("handoff-verdict",
              str(_verdict.get("verdict") or "?") + " — "
              + str(_verdict.get("reason") or ""))
    except Exception:
        pass

    # 0.2.0e Build 11 (#198): the homecoming adoption. When the
    # installer migrated Apex to its new home, it left a machine
    # marker in this (now parked) tree naming the new home. When
    # the new home verifiably holds THIS job's target build,
    # everything from here — the landed classification below,
    # the wedge clear, the relaunch, the confirmation, and the
    # record itself — follows the install to its new home
    # instead of reading the parked tree's old witnesses.
    try:
        with open(os.path.join(root, ".apex-migration.json")) as _mf:
            _mig = json.load(_mf)
        _new_root = str(_mig.get("to") or "")
        if (_new_root and _new_root != root
                and os.path.isfile(os.path.join(_new_root, "bridge.py"))
                and read_installed_build_seq(_new_root)
                == job.get("targetBuildSeq")):
            trace("migration-adopted",
                  "the installer moved Apex to " + _new_root
                  + " — relaunch, confirmation and the record follow it there")
            root = _new_root
            STATE["result_path"] = os.path.join(
                root, "config", "update-result.json")
            env["NAVI_HOME"] = root
    except Exception:
        pass

    # v0.4.3e (#237): the stage comes home BEFORE any verdict
    # or record close below — the shell verdict's failure text
    # and every later failure text name the install log and
    # the staged package at their live paths, so those paths
    # are made true first (see _recover_stage_artifacts).
    _recover_stage_artifacts(job, _log_handle)
    if _log_handle is not None:
        try:
            _log_handle.close()
        except Exception:
            pass
        _log_handle = None

    # v0.4.6e (#185): the headless gate on the shell verdicts.
    # On a headless install the shell build was SKIPPED BY
    # DETERMINATION — there is no binary beside start.sh, and
    # none is owed. Both verdicts below must therefore never
    # fire here: the disk verdict (_shell_window_state) would
    # read the missing binary as the designed browser fallback
    # only by luck of its no-binary branch, and the #237
    # owed-but-missing check reads the install log — a log the
    # headless installer fills with its own named skip line,
    # not the owed marker, but the gate does not rest on that
    # courtesy. The skip is NAMED, in the trace and in the
    # apply record, as the successful step it is: the window
    # step of a headless apply is "skipped-headless", decided
    # before either verdict is consulted. The determination is
    # re-read NOW, after the installer ran, so the installed
    # config's recorded verdict for this very install (the
    # stamp the installer just wrote) is the first witness.
    _headless_final, _hl_final_source = _install_headless(job, root)
    _shell_state = None
    if _headless_final:
        _shell_state = "skipped-headless"
        trace("shell-window",
              "skipped-headless — the shell build was skipped "
              "by determination on this headless install "
              "(#185): " + _hl_final_source + ". No window is "
              "built or expected, so neither the disk verdict "
              "nor the owed-but-missing check applies.")
        write_result({"headless": True,
                      "shellWindow": "skipped-headless"})

    # v0.4.2e (#233): the shell-window verdict, read from disk
    # (see _shell_window_state). An apply is NOT confirmed
    # while the installed window binary is one the launcher
    # will refuse: v0.4.1e's pull laid new shell sources over a
    # binary built from older ones, the install reported
    # success, and the operator learned it at the next launch,
    # from a launcher log. A stale verdict fails the apply
    # HERE — in the record the panel shows, with the reason in
    # words — whatever the installer's exit code claimed.
    if _shell_state is None:
        _shell_state, _shell_reason = _shell_window_state(root)
    else:
        _shell_reason = ""
    if _shell_state != "skipped-headless":
        trace("shell-window",
              _shell_state
              + ((" — " + _shell_reason) if _shell_reason else ""))
    if _shell_state == "stale":
        fail("install",
             "The update installed, but her standalone window "
             "did not come with it: " + _shell_reason + ". The "
             "update is not confirmed. The installer's shell "
             "build log is at shell/shell-build.log in the "
             "install folder — the Rust toolchain and the "
             "WebKitGTK development packages are the usual "
             "causes. Fix the cause and run the update again, "
             "or use the manual fallback below, which re-runs "
             "its installer by hand.")
        relaunch()
        STATE["stop_beat"] = True
        return

    # v0.4.3e (#237): the owed-but-missing window. The disk
    # verdict above reads "ok" when no binary stands beside
    # start.sh — the designed browser fallback — but on an
    # update whose shell sources CHANGED, a build was owed,
    # and a build that failed retires the stale binary into
    # exactly that no-binary shape. The installer names the
    # distinction in its own log (SHELL-WINDOW:
    # owed-but-missing); when it does, the absence is a
    # missing window, not a fallback, and the apply fails on
    # it here the same way a stale verdict does.
    if _shell_state == "ok":
        _owed_missing = False
        try:
            with open(job["installLog"], "rb") as _lf:
                _owed_missing = (
                    b"SHELL-WINDOW: owed-but-missing" in _lf.read())
        except Exception:
            _owed_missing = False
        if _owed_missing:
            trace("shell-window",
                  "owed-but-missing — the shell sources changed "
                  "with this update and the build produced no "
                  "window; the apply fails on it in its record")
            fail("install",
                 "The update installed, but her standalone window "
                 "did not come with it: the shell sources changed "
                 "with this update and the build did not produce "
                 "a window — the previous binary was retired "
                 "rather than left standing stale. The update is "
                 "not confirmed. The installer's shell build log "
                 "is at shell/shell-build.log in the install "
                 "folder — the Rust toolchain and the WebKitGTK "
                 "development packages are the usual causes. Fix "
                 "the cause and run the update again, or use the "
                 "manual fallback below, which re-runs its "
                 "installer by hand.")
            relaunch()
            STATE["stop_beat"] = True
            return

    tail_failure = ""
    if code != 0:
        # Issue #104 (Build 6): a nonzero exit is not the whole
        # story, in either direction. Classify before reporting:
        # (2) the exit was nonzero BUT the target build is already
        # installed on disk — the installer's version flip is its
        # final act (Build 6), so a matching VERSION file and a
        # matching baked-in build sequence together prove the new
        # build landed and only a tail step failed. That is NOT
        # the manual-fallback failure: fall through to the normal
        # restart + confirmation with the tail failure recorded,
        # and let confirmation have the last word. (3) Anything
        # else is the original failure, unchanged.
        tgt_seq_chk = job.get("targetBuildSeq")
        chk_number, _chk_status = read_installed_version(root)
        chk_seq = read_installed_build_seq(root)
        installed_ok = bool(
            chk_number
            and tuples_equal(
                version_tuple(chk_number), tuple(job["targetTuple"]))
            and (tgt_seq_chk is None or chk_seq == tgt_seq_chk))
        if installed_ok:
            tail_failure = _install_log_last_step(job["installLog"])
            trace("installer-tail-failure",
                  "exit " + str(code) + " but the target build is "
                  "installed (version " + chk_number + ", buildSeq "
                  + str(chk_seq) + "); tail step: " + tail_failure)
        else:
            reason = why or ("it exited with code " + str(code))
            fail("install",
                 "Installing the update failed — " + reason + ". The "
                 "installer's own output is in updates/" + job["tagDir"]
                 + "/apply-install.log. Nothing is confirmed updated: "
                 "the app is being relaunched as it lies, and the "
                 "staged package is still in place — the manual "
                 "fallback below finishes the job by hand.")
            relaunch()
            STATE["stop_beat"] = True
            return

    # The installer's DHD prerequisite step is a named ladder
    # step of its own: its summary line is read back from the
    # install log the helper just captured, so the panel shows
    # the step because the installer really ran it — never as a
    # file-copy shortcut or a relabelled install phase.
    dhd_prereq_detail = ""
    try:
        with open(job["installLog"], "rb") as log:
            install_text = log.read().decode("utf-8", "replace")
        for line in install_text.splitlines():
            if line.startswith("DHD-PREREQS:"):
                dhd_prereq_detail = line.split(":", 1)[1].strip()
                break
    except Exception:
        dhd_prereq_detail = ""
    trace("dhd-prereqs",
          dhd_prereq_detail or "no DHD-PREREQS line in the install log")
    STATE["step"] = "dhd-prereqs"
    write_result({
        "state": "working", "step": "dhd-prereqs",
        "dhdPrereqs": dhd_prereq_detail or "This installer did not report a DHD prerequisite step.",
    })
    time.sleep(1)
    STATE["step"] = "restarting"
    write_result({"state": "working", "step": "restarting"})
    # Build .91 (issue #91): prove the old bridge is gone BEFORE
    # relaunching. A wedged holder makes every relaunch a no-op —
    # the new bridge cannot bind — and the old failure text's
    # remedy ("launch the app again") cannot work while the port
    # is held. Escalate here exactly as the installer does; if
    # the port still cannot be cleared, fail naming the holder
    # and the exact command that ends it.
    wedge_ok, wedge_pid, wedge_desc = clear_wedged_bridge(
        root, job["port"], job.get("targetBuildSeq"))
    if not wedge_ok:
        if wedge_pid:
            wedge_remedy = (
                "End it with: kill -9 " + str(wedge_pid)
                + " — then run the update again, or the manual "
                "fallback below. Launching the app again cannot "
                "work while that process holds the port.")
        else:
            wedge_remedy = (
                "Find what holds it with: lsof -i :" + str(job["port"])
                + " — end that process, then run the update again, "
                "or the manual fallback below. Launching the app "
                "again cannot work while the port is held.")
        fail("restart",
             "The update installed, but the old bridge could not "
             "be cleared from http://localhost:" + str(job["port"])
             + ": " + wedge_desc + ". The new bridge cannot start "
             "while the port is held, so the update is not "
             "confirmed. " + wedge_remedy
             + " The staged package is still in place.")
        STATE["stop_beat"] = True
        return
    relaunch_err = relaunch()
    if relaunch_err:
        if _port_answers_expected(job["port"], job.get("targetBuildSeq")):
            # The spawn failed but the target build is already
            # answering — the installer's own relaunch (build
            # .91) brought her up. Not a failure: fall through to
            # the confirmation loop, which will see her itself.
            trace("relaunch-skipped",
                  "spawn failed (" + relaunch_err + ") but the target build already answers")
            relaunch_err = ""
        else:
            fail("restart",
                 "The update installed, but relaunching Apex through "
                 "start.sh could not even be started (" + relaunch_err
                 + "), so the update is not confirmed. Expected "
                 + job["targetLabel"]
                 + (" Build " + str(job["targetBuildSeq"])
                    if job.get("targetBuildSeq") is not None else "")
                 + " answering on http://localhost:" + str(job["port"])
                 + "; found: no relaunch was started. The staged "
                 "package is still in place — the manual fallback "
                 "below finishes the job by hand, or launch the app "
                 "again from its menu entry.")
            STATE["stop_beat"] = True
            return
    target = tuple(job["targetTuple"])
    # Build 4: when the job names a target Build number, the
    # confirmation must observe that build answering — the
    # version number is identical on both sides of a refresh,
    # so a version-only match once certified the previous
    # build's reinstall as the new build's arrival.
    tgt_seq = job.get("targetBuildSeq")
    answered = None
    last_seen = ""
    deadline = time.time() + 300
    while time.time() < deadline:
        seen = None
        try:
            req = urllib.request.Request(
                "http://127.0.0.1:" + str(job["port"]) + "/api/status",
                headers={"User-Agent": "navi-update-helper"})
            with urllib.request.urlopen(req, timeout=3) as resp:
                seen = json.loads(resp.read().decode("utf-8", "replace"))
        except Exception:
            seen = None
        if seen:
            last_seen = (
                "app=" + str(seen.get("app"))
                + " version=" + str(seen.get("version"))
                + " buildSeq=" + str(seen.get("buildSeq")))
        if seen and seen.get("app") == "Navi: Apex":
            if answered is None:
                trace("bridge-answered", last_seen)
            answered = seen
            if (seen.get("version")
                    and tuples_equal(
                        version_tuple(seen["version"]), target)
                    and (tgt_seq is None
                         or seen.get("buildSeq") == tgt_seq)):
                break
        time.sleep(2)
    after_number, after_status = read_installed_version(root)
    after_label = (
        version_label(after_number, after_status) if after_number else "")
    disk_seq = read_installed_build_seq(root)
    trace("confirm",
          "live: " + (last_seen or "no answer")
          + "; on disk: version=" + (after_number or "(unreadable)")
          + " buildSeq=" + str(disk_seq)
          + "; expected: " + job["targetLabel"]
          + " buildSeq=" + str(tgt_seq))
    live_ok = bool(
        answered and answered.get("version")
        and tuples_equal(version_tuple(answered["version"]), target)
        and (tgt_seq is None or answered.get("buildSeq") == tgt_seq))
    file_ok = bool(
        after_number
        and tuples_equal(version_tuple(after_number), target)
        and (tgt_seq is None or disk_seq is None
             or disk_seq == tgt_seq))
    if live_ok and file_ok:
        STATE["step"] = "done"
        write_result({
            "state": "done", "step": "done",
            **({"tailFailure": tail_failure} if tail_failure else {}),
            "afterVersion": after_number,
            "afterStatus": after_status,
            # The worker's composed to-side label when it sent
            # one (Build 3: a refresh's "Build <n>"); the label
            # read from the VERSION file otherwise — identical
            # for graduations and new versions.
            "afterLabel": job.get("afterLabel") or after_label,
            "updatedAt": time.strftime(
                "%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        })
        trace("done",
              "confirmed " + str(job.get("afterLabel") or after_label))
    elif answered is None:
        fail("restart",
             "The update installed, but Apex did not answer on "
             "http://localhost:" + str(job["port"]) + " after "
             "restarting, so the update is not confirmed. Expected "
             + job["targetLabel"]
             + (" Build " + str(tgt_seq) if tgt_seq is not None else "")
             + " answering there; found: "
             + (("an answer that was not the expected bridge ("
                 + last_seen + ")")
                if last_seen else
                "no answer at all in five minutes")
             + ". "
             + _holder_remedy(job["port"])
             + " The staged package is still in place — the manual "
             "fallback below re-runs its installer by hand.")
    elif (tgt_seq is not None and disk_seq is not None
            and disk_seq != tgt_seq):
        fail("restart",
             "Apex came back after the update, but the bridge "
             "installed on disk is Build " + str(disk_seq)
             + " instead of the expected Build " + str(tgt_seq)
             + " — the update is not confirmed. The staged "
             "package is still in place — the manual fallback "
             "below finishes the job by hand.")
    elif (tgt_seq is not None and answered is not None
            and answered.get("buildSeq") is not None
            and answered.get("buildSeq") != tgt_seq):
        fail("restart",
             "Apex came back after the update, but the bridge "
             "answering reports Build "
             + str(answered.get("buildSeq"))
             + " instead of the expected Build " + str(tgt_seq)
             + " — the update is not confirmed. "
             + _holder_remedy(job["port"])
             + " The staged package is still in place — the manual "
             "fallback below finishes the job by hand.")
    else:
        fail("restart",
             "Apex came back after the update, but its VERSION "
             "file reads " + (after_label or "nothing readable")
             + " instead of the expected " + job["targetLabel"]
             + " — the update is not confirmed. The staged "
             "package is still in place — the manual fallback "
             "below finishes the job by hand.")
    STATE["stop_beat"] = True


def _guarded_main():
    # The top-level guard: any Python-level failure anywhere in
    # the apply — the job load, a record write, the relaunch,
    # the confirmation — lands in the step log with its
    # traceback, in the record as a failed state naming the
    # step it died on and the reason in words, and on stderr
    # (which the bridge points at apply.log, so that file is
    # never again 0 bytes on a death words could describe).
    try:
        main()
    except BaseException as e:
        tb = traceback.format_exc()
        trace("fatal", tb.replace("\\n", " | ")[-1500:])
        try:
            print(tb, file=sys.stderr)
            sys.stderr.flush()
        except Exception:
            pass
        job = STATE.get("job") or {}
        try:
            write_result({
                "state": "failed", "step": STATE.get("step"),
                "error": (
                    "The apply helper failed at the "
                    + str(STATE.get("step")) + " step: "
                    + type(e).__name__ + ": " + str(e)
                    + " — the update is not confirmed. Its step "
                    "log (apply-trace.log, beside the staged "
                    "package) carries the full trace. The staged "
                    "package is still in place — the manual "
                    "fallback below finishes the job by hand."),
                "stagedDir": job.get("pkgDir", ""),
                "command": job.get("command", ""),
                "failedAt": _stamp(),
            })
        except Exception:
            pass
        sys.exit(1)


if __name__ == "__main__":
    _guarded_main()
'''


_update_apply_lock = threading.Lock()
_update_apply_state = {"running": False}


def _apply_helper_spawn_argv(helper_path, job_path, tag_dir):
    # Build 12 (#201) — the helper escape, and the invariant it
    # serves. THE NO-ANCESTRY-SIGNALLING INVARIANT: an apply may
    # end exactly one process by signal — the bridge it replaces,
    # by its proven pid, through the installer's stop machinery.
    # It may never use a group-shaped signal — a systemd unit
    # stop, a process-group kill, a cgroup kill — because those
    # mechanisms cannot tell the bridge apart from the apply's
    # own ancestry: the helper descends from the bridge, and the
    # installer descends from the helper. Incident 21 is the
    # tuition: v0.2.0e — Build 11's first unattended pull died
    # within two seconds on both field machines when the
    # installer's homecoming retired the bridge's boot unit with
    # `systemctl --user disable --now` — stopping the unit
    # signalled every process in it (the bridge, the helper and
    # the installer together; a new session is not an escape
    # from a cgroup), and the migration never ran.
    #
    # The installer's half of the fix is ordering: it stops the
    # bridge by pid and retires the unit files without stopping
    # anything. This is the bridge's half: the helper is spawned
    # OUTSIDE this bridge's systemd unit, in a transient scope
    # of its own via systemd-run, so no stop of the bridge's
    # unit — this installer's, a future installer's, an
    # operator's — can ever reach it. In scope mode systemd-run
    # leaves a waiting parent whose lifetime tracks the
    # helper's, so the pid recorded on the apply record keeps
    # its meaning for the panel and the watchdog. Where no
    # scope can be had (no systemd-run, or no user manager
    # answers — the Docker shape), the spawn falls back to the
    # plain own-session spawn and SAYS SO, in the step log and
    # on the record: on such a machine the installer's ordering
    # is the protection, and nothing pretends otherwise.
    # Returns (argv, mode, note).
    base = [sys.executable, helper_path, job_path]
    sd = shutil.which("systemd-run")
    if sd:
        probe = None
        try:
            probe = subprocess.run(
                [sd, "--user", "--scope", "--collect", "-q",
                 "/bin/true"],
                stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL,
                stderr=subprocess.DEVNULL, timeout=10)
        except Exception:
            probe = None
        if probe is not None and probe.returncode == 0:
            unit = ("apex-apply-"
                    + re.sub(r"[^0-9A-Za-z_-]+", "-", str(tag_dir))
                    + "-" + str(os.getpid()))
            argv = [sd, "--user", "--scope", "--collect", "-q",
                    "--unit=" + unit, "--"] + base
            return (argv, "systemd-run-scope",
                    "the apply helper runs in its own transient "
                    "scope (" + unit + ".scope), outside this "
                    "bridge's systemd unit — a stop of the "
                    "bridge's unit during the install cannot "
                    "signal it (#201).")
    return (base, "plain-session",
            "systemd-run could not give the apply helper a scope "
            "of its own here (not installed, or no user manager "
            "answers), so it runs plainly in its own session, "
            "inside this bridge's unit — the installer's "
            "ordering is this run's protection: it stops the "
            "bridge by pid and never stops its unit (#201).")


def _run_update_apply(cfg):
    # The apply worker (build .56): download -> verify -> stage on
    # this thread, writing each step into the update record, then
    # hand the install itself to the detached helper and finish.
    # Every failure lands in the record with its step named; a
    # failure before staging completes carries no fallback (there
    # is no complete staged package to fall back to), and nothing
    # is extracted before the package verifies.
    record = {
        "state": "working", "step": "downloading",
        "startedAt": _now_iso(),
    }
    staged = {"dir": "", "command": ""}

    def fail(step, error):
        record["state"] = "failed"
        record["step"] = step
        record["error"] = error
        record["failedAt"] = _now_iso()
        if staged["dir"]:
            record["stagedDir"] = staged["dir"]
            record["command"] = staged["command"]
        _write_update_result(record)

    try:
        current = app_version(cfg)
        current_status = app_version_status(cfg)
        record["beforeVersion"] = current
        record["beforeStatus"] = current_status
        record["beforeLabel"] = _version_label(current, current_status)
        try:
            rel = latest_release(updater_channel(cfg))
        except Exception as e:
            fail("downloading",
                 "Downloading the update failed — the release check "
                 "did not complete: " + _update_error_text(e))
            return
        tag = rel["tag"]
        t_num, t_status = _tag_parts(tag)
        record["tag"] = tag
        record["targetLabel"] = (
            _version_label(t_num, t_status) if t_num else tag)
        # Build 3: a same-version refresh — same numbers, same
        # status, a higher Build number — names the builds in its
        # labels, because the plain version labels read as no
        # change at all ("Updated v0.1.8e → v0.1.8e"). The pair
        # reads "v0.1.8e Build 2 → Build 3": the from-side keeps
        # its version (it says what was running), the to-side is
        # just the build it lands on. Graduation and new-version
        # updates keep the plain version labels, untouched.
        _tgt_seq = rel.get("buildSeq")
        if t_num and current_status == "edge" and _tgt_seq is not None                 and _tgt_seq > SHIPPED_BUILD_SEQ:
            _a, _b = _version_tuple(t_num), _version_tuple(current)
            _w = max(len(_a), len(_b))
            _same_numbers = (_a + (0,) * (_w - len(_a))) == (
                _b + (0,) * (_w - len(_b)))
            if _same_numbers and _status_rank(t_status) == _status_rank(
                    current_status):
                record["beforeLabel"] = (
                    record["beforeLabel"] + " Build "
                    + str(SHIPPED_BUILD_SEQ))
                record["targetLabel"] = "Build " + str(_tgt_seq)
        if not tag or not _release_is_newer(tag, current, current_status, rel.get("buildSeq")):
            fail("downloading",
                 "The latest release (" + (tag or "none") + ") is not "
                 "newer than the running v" + current
                 + " — nothing was downloaded.")
            return
        asset = rel["asset"]
        if not asset or not asset["url"]:
            fail("downloading",
                 "The latest release carries no package .zip asset, "
                 "so there is nothing to download.")
            return
        record["assetName"] = asset["name"]
        tag_dir = re.sub(r"[^0-9A-Za-z._-]+", "-", tag) or "release"
        stage_root = os.path.join(UPDATES_DIR, tag_dir)
        pkg_dir = os.path.join(stage_root, "package")
        zip_path = os.path.join(stage_root, asset["name"])
        # The manual fallback carries the install's known answers
        # as flags (build .84, issue #75): the fallback gets run by
        # hand exactly where prompts go to die — shells with no
        # usable terminal, the app's own CLI shell first among
        # them — so it must never need to ask a question. Startup
        # comes from the config; voice counts as installed when her
        # venv Python is on disk (the installer still verifies the
        # engines import before it skips the rebuild). The apply
        # job below carries the same two answers.
        fb_cfg = cfg if isinstance(cfg, dict) else {}
        fb_startup = (
            "boot"
            if str(fb_cfg.get("startup") or "on-ui-launch") == "on-boot"
            else "launch")
        fb_voice = (
            "yes"
            if os.path.isfile(os.path.join(
                ROOT, "voice", "venv", "bin", "python"))
            else "no")
        command = (
            "cd " + pkg_dir + " && bash install.sh --startup "
            + fb_startup + " --voice " + fb_voice)
        # Downloading (Build 4 — the refresh-identity repair).
        # A package already downloaded beside the app — including
        # one a pre-.56 Updater refused after downloading — is
        # read back instead of being fetched again ONLY when it
        # is provably the package the feed is offering now: its
        # byte size must equal the feed asset's size. The staging
        # folder is named by the release TAG and the file by the
        # asset name, and both are identical for every
        # publication of one version — so a same-version refresh
        # otherwise finds the PREVIOUS build's zip sitting at
        # this exact path (left there by that build's own apply)
        # and installs those bytes again. That is the Build 2 →
        # Build 3 field failure: the apply ran clean, the
        # checklist passed, and the install still identified as
        # Build 2 afterwards, because the bytes it installed WERE
        # Build 2's — and the version-only confirmation below
        # could not tell the difference. A size mismatch (or a
        # feed that names no size) fetches fresh, overwriting
        # the leftover; the build-identity check in verification
        # is the second lock on the same door, for a leftover
        # whose size happens to match.
        feed_seq = rel.get("buildSeq")
        feed_size = 0
        try:
            feed_size = int(asset.get("size") or 0)
        except Exception:
            feed_size = 0
        _write_update_result(record)
        data = None
        from_disk = False
        if os.path.isfile(zip_path):
            try:
                if feed_size and os.path.getsize(zip_path) == feed_size:
                    with open(zip_path, "rb") as f:
                        data = f.read()
                    from_disk = True
            except Exception as e:
                fail("downloading",
                     "Downloading the update failed: the package "
                     "already beside the app could not be read ("
                     + str(e) + ") — nothing was staged.")
                return
        if data is None:
            try:
                data = github_download(asset["url"])
            except Exception as e:
                fail("downloading",
                     "Downloading the update failed ("
                     + type(e).__name__ + ") — nothing was staged.")
                return
        # Verifying: unwrap a gzip-wrapped payload, then the same
        # Apex-package verification as always — a real zip, holding
        # install.sh + VERSION, with no path escaping its folder —
        # plus the build-identity check (Build 4): when the feed
        # names the target's Build number, the package in hand
        # must BE that build; its bridge.py carries the sequence
        # the installed app will report. A package naming a
        # different build is the wrong bytes, however valid a zip
        # it is. When the wrong bytes came from the leftover
        # beside the app, they are discarded and the offered
        # package is fetched fresh, once, and verified again;
        # when a fresh download itself names the wrong build, the
        # feed and its asset disagree and the apply fails here,
        # in words, with nothing staged.
        record["step"] = "verifying"
        _write_update_result(record)
        retried_fresh = False
        while True:
            payload, _note = _unwrap_package(data)
            if payload is None:
                if _note == "gzip-broken":
                    fail("verifying",
                         "The download arrived gzip-wrapped but "
                         "could not be unwrapped — nothing was "
                         "staged.")
                else:
                    fail("verifying",
                         "The download arrived gzip-wrapped, but "
                         "what is inside is not a zip file — "
                         "nothing was staged.")
                return
            try:
                zf = zipfile.ZipFile(io.BytesIO(payload))
                names = zf.namelist()
            except Exception:
                fail("verifying",
                     "The download did not arrive as a zip file "
                     "— nothing was staged.")
                return
            if "install.sh" not in names or "VERSION" not in names:
                fail("verifying",
                     "That zip is not an Apex package (no "
                     "install.sh / VERSION inside) — nothing was "
                     "staged.")
                return
            base = os.path.realpath(pkg_dir)
            escaped = ""
            for member in names:
                target = os.path.realpath(
                    os.path.join(pkg_dir, member))
                if target != base and not target.startswith(
                        base + os.sep):
                    escaped = member
                    break
            if escaped:
                fail("verifying",
                     "The package zip contains a path outside its "
                     "folder (" + escaped + ") — nothing was "
                     "staged.")
                return
            if feed_seq is not None and "bridge.py" in names:
                pkg_seq = None
                try:
                    with zf.open("bridge.py") as bf:
                        _m = re.search(
                            rb"SHIPPED_BUILD_SEQ\s*=\s*([0-9]+)",
                            bf.read())
                    if _m:
                        pkg_seq = int(_m.group(1))
                except Exception:
                    pkg_seq = None
                if pkg_seq is not None and pkg_seq != feed_seq:
                    if from_disk and not retried_fresh:
                        retried_fresh = True
                        from_disk = False
                        try:
                            data = github_download(asset["url"])
                        except Exception as e:
                            fail("downloading",
                                 "Downloading the update failed ("
                                 + type(e).__name__ + ") — nothing "
                                 "was staged.")
                            return
                        continue
                    fail("verifying",
                         "The package in hand is Build "
                         + str(pkg_seq) + ", but the release on "
                         "offer is Build " + str(feed_seq)
                         + " — not the same bytes, so nothing was "
                         "staged. Check again to fetch the offered "
                         "package.")
                    return
            break
        # Staging: the verified (unwrapped) zip lands beside the
        # app and unpacks under updates/<tag>/package, as before.
        record["step"] = "staging"
        _write_update_result(record)
        try:
            os.makedirs(pkg_dir, exist_ok=True)
            with open(zip_path, "wb") as f:
                f.write(payload)
            zf.extractall(pkg_dir)
            try:
                os.chmod(os.path.join(pkg_dir, "install.sh"), 0o755)
            except Exception:
                pass
        except Exception as e:
            fail("staging", "Staging the download failed: " + str(e))
            return
        staged["dir"] = pkg_dir
        staged["command"] = command
        record["stagedDir"] = pkg_dir
        record["command"] = command
        record["step"] = "installing"
        _write_update_result(record)
        # Installing + restarting belong to the detached helper:
        # the installer stops this bridge, so the bridge cannot
        # run it and live to tell the panel. The helper outlives
        # the swap, relaunches the app, and finishes the record.
        try:
            cfg_now = cfg if isinstance(cfg, dict) else load_config()
            job = {
                "root": ROOT,
                "pkgDir": pkg_dir,
                "tagDir": tag_dir,
                "resultPath": UPDATE_RESULT_PATH,
                "installLog": os.path.join(
                    stage_root, "apply-install.log"),
                "relaunchLog": os.path.join(
                    stage_root, "apply-relaunch.log"),
                # Build 6: the helper's step log — written from
                # its first instruction, beside the job, with its
                # path carried on the apply record so the panel,
                # the watchdog, and anyone reading the wreckage
                # afterwards can find the last step it completed.
                "traceLog": os.path.join(
                    stage_root, "apply-trace.log"),
                "command": command,
                "port": BRIDGE_PORT,
                "startup": str(cfg_now.get("startup") or "on-ui-launch"),
                "voice": fb_voice,
                "targetTuple": list(_version_tuple(tag)),
                # Build 4: the target's Build number, when the
                # feed names one — the helper's confirmation must
                # observe THIS landing, not just the version
                # number a refresh shares with its predecessor.
                "targetBuildSeq": rel.get("buildSeq"),
                "targetLabel": record["targetLabel"],
                # Build 3: the label the finished record carries
                # for the to-side — composed here, where the
                # target's Build number is known (for a refresh
                # it is "Build <n>", which the helper cannot
                # derive from the VERSION file it confirms
                # against; the plain version label otherwise).
                "afterLabel": record["targetLabel"],
            }
            job_path = os.path.join(stage_root, "apply-job.json")
            helper_path = os.path.join(stage_root, "apply-helper.py")
            # v0.3.0e Build 1 (#205, the bootstrap patch): when
            # the staged package carries its own apply machinery
            # (apply-bootstrap.py + apply-helper.py beside the
            # payload), the PACKAGE's bootstrap spawns the
            # package's helper — the apply is governed by the
            # incoming build's machinery from its first step, not
            # by this bridge's spawner, whose protection a pull
            # can never inherit for the pull that installs it
            # (incident 22). The job carries the bootstrap flag,
            # so the helper marks the installer's environment and
            # the installer's own handoff stays out of the way.
            pkg_boot = os.path.join(pkg_dir, "apply-bootstrap.py")
            pkg_helper = os.path.join(pkg_dir, "apply-helper.py")
            use_bootstrap = (os.path.isfile(pkg_boot)
                             and os.path.isfile(pkg_helper))
            boot_ok = False
            boot_mode = ""
            boot_note = ""
            boot_pid = None
            if use_bootstrap:
                job["bootstrap"] = True
                job["spawnMode"] = "package-bootstrap"
                with open(job_path, "w") as f:
                    json.dump(job, f)
                try:
                    br = subprocess.run(
                        [sys.executable, pkg_boot, job_path],
                        capture_output=True, text=True, timeout=90)
                    verdict = json.loads(
                        (br.stdout or "").strip().splitlines()[-1])
                    boot_ok = bool(verdict.get("ok"))
                    boot_mode = str(verdict.get("mode") or "")
                    boot_note = str(verdict.get("note") or "")
                    boot_pid = verdict.get("helperPid")
                except Exception as e:
                    boot_ok = False
                    boot_note = ("the package bootstrap could not "
                                 "run: " + str(e))
                if boot_ok:
                    try:
                        with open(job["traceLog"], "a") as tf:
                            tf.write(time.strftime(
                                "%Y-%m-%dT%H:%M:%SZ", time.gmtime())
                                + " [starting] helper-spawn — the "
                                "staged package's bootstrap spawned "
                                "its own helper (" + boot_mode
                                + "): " + boot_note + "\n")
                    except Exception:
                        pass
                    record["helperPid"] = boot_pid
                    record["helperSpawn"] = ("package-bootstrap/"
                                              + boot_mode)
                    record["traceLog"] = job["traceLog"]
                    _write_update_result(record)
            if not boot_ok:
                # The legacy path — and the bootstrap's failure
                # fallback, named as such on the record. Build 12
                # (#201): decide HOW the helper is spawned before
                # the job is written — the mode rides the job and
                # the record, and is announced as the step log's
                # first line, ahead of the helper's own first
                # entry, so the trace always says which
                # protection this run actually has (see
                # _apply_helper_spawn_argv).
                spawn_argv, spawn_mode, spawn_note = \
                    _apply_helper_spawn_argv(helper_path, job_path, tag_dir)
                if use_bootstrap:
                    spawn_note = ("the package bootstrap did not "
                                  "arm (" + boot_note + ") — this "
                                  "bridge's spawner carries the "
                                  "apply instead: " + spawn_note)
                    # job["bootstrap"] stays set: the embedded
                    # helper is the package's own machinery
                    # (byte-identical), so the installer's
                    # handoff must stay out of its way too.
                job["spawnMode"] = spawn_mode
                with open(job_path, "w") as f:
                    json.dump(job, f)
                with open(helper_path, "w") as f:
                    f.write(APPLY_HELPER_PY)
                os.chmod(helper_path, 0o755)
                try:
                    with open(job["traceLog"], "a") as tf:
                        tf.write(time.strftime(
                            "%Y-%m-%dT%H:%M:%SZ", time.gmtime())
                            + " [starting] helper-spawn — " + spawn_note
                            + "\n")
                except Exception:
                    pass
                logf = open(os.path.join(stage_root, "apply.log"), "ab")
                try:
                    proc = subprocess.Popen(
                        spawn_argv,
                        cwd=stage_root, stdin=subprocess.DEVNULL,
                        stdout=logf, stderr=subprocess.STDOUT,
                        start_new_session=True, close_fds=True)
                finally:
                    logf.close()
                record["helperPid"] = proc.pid
                record["helperSpawn"] = spawn_mode
                record["traceLog"] = job["traceLog"]
                _write_update_result(record)
        except Exception as e:
            fail("install",
                 "The update was staged but the install could not "
                 "be started: " + str(e) + " The staged package is "
                 "still in place — the manual fallback below "
                 "finishes the job by hand.")
            return
    except Exception as e:
        # Last resort: never leave the record claiming a step is
        # still running when this worker has died.
        try:
            record["state"] = "failed"
            record["error"] = "The update failed: " + str(e)
            record["failedAt"] = _now_iso()
            _write_update_result(record)
        except Exception:
            pass
    finally:
        _update_apply_state["running"] = False


def update_apply_payload(cfg=None):
    # Starts the end-to-end apply (see the section comment) and
    # answers at once: progress and outcome live in the update
    # record, which GET /api/update/result serves and the panel
    # polls. A second apply while one is already working — this
    # bridge's worker, or the detached helper a previous click
    # left installing — is refused in words, never queued.
    rec = update_result_payload()
    helper_alive = False
    if rec.get("state") == "working" and rec.get("helperPid"):
        try:
            helper_alive = pid_alive(int(rec["helperPid"]))
        except Exception:
            helper_alive = False
    with _update_apply_lock:
        if _update_apply_state["running"] or helper_alive:
            return {"ok": False, "error": "An update is already "
                    "being applied — watch its progress in the "
                    "Updater; it finishes (or fails) on its own."}
        _update_apply_state["running"] = True
    t = threading.Thread(
        target=_run_update_apply, args=(cfg,), daemon=True)
    t.start()
    return {"ok": True, "started": True}


def save_download_payload(data):
    # The dashboard's download-save flow (build .48, issue #10):
    # inside the desktop shell the webview cannot save a file the
    # way a browser does, so when the operator clicks a download
    # there (a chip .png, an export ZIP) the dashboard fetches the
    # bytes itself and hands them here as base64; the bridge writes
    # them into the operator's Downloads folder. The filename is
    # never trusted as a path: it is reduced to a plain basename,
    # a name already taken gains a numeric suffix instead of an
    # overwrite, and anything over the cap is refused outright.
    name = str(data.get("filename") or "").strip()
    name = name.replace(chr(92), "/").split("/")[-1].strip()
    if name in ("", ".", ".."):
        name = "navi-download"
    try:
        blob = base64.b64decode(str(data.get("dataBase64") or ""),
                                validate=True)
    except Exception:
        return {"ok": False, "error": "That download did not arrive "
                "as readable data — nothing was saved."}
    if len(blob) > 64 * 1024 * 1024:
        return {"ok": False, "error": "That file is larger than the "
                "64 MB download limit — nothing was saved."}
    try:
        folder = os.path.join(os.path.expanduser("~"), "Downloads")
        os.makedirs(folder, exist_ok=True)
        target = os.path.join(folder, name)
        if os.path.exists(target):
            stem, dot, ext = name.rpartition(".")
            n = 2
            while os.path.exists(target):
                if stem and dot:
                    candidate = stem + " (" + str(n) + ")." + ext
                else:
                    candidate = name + " (" + str(n) + ")"
                target = os.path.join(folder, candidate)
                n += 1
        with open(target, "wb") as f:
            f.write(blob)
    except Exception as e:
        return {"ok": False, "error": "Saving to the Downloads "
                "folder failed: " + str(e)}
    return {"ok": True, "path": target, "sizeBytes": len(blob)}


# ---------------------------------------------------------------------
# GitHub: optional sign-in (Device Flow) and backup/import of the
# ready-to-fly essentials (build .35).
#
# Sign-in is OPTIONAL and feature-scoped: Navi runs fully without it —
# chat, chips, RAG, offline. It exists only for GitHub-backed actions
# (backup/import here). The flow is GitHub's Device Flow: the bridge
# asks GitHub for a user code, the operator authorizes in their own
# browser at github.com/login/device, and the bridge polls the token
# endpoint. No client secret is embedded and no password ever passes
# through the app. The OAuth client id is a public identifier: the
# package ships the project's registered id in
# config/github-signin.json (Build 6, #173 phase 3), so sign-in
# works out of the box — one credential for the whole product.
#
# The token lands in config/github-auth.json with owner-only
# permissions. It is NEVER part of a backup (a backup gathers named
# files; this is not one of them), never logged, and signing out
# deletes the file.
#
# A backup is exactly the ready-to-fly essentials, written as files
# into a repo under the operator's own account (default navi-backup,
# created private unless they chose public):
#   navi-backup.json    the manifest (format navi-backup/1)
#   soul.md             her Soul document
#   memory.md           her Memory document
#   settings.json       the app config minus the github section, the
#                       bridge block (the port is a build property),
#                       and the hardware profile + runtime flags (a
#                       fresh install profiles its own machine; an
#                       imported profile must never override that)
#   voice.json          her voice choice and voice settings
#   dashboard.json      the settings that live in the dashboard
#                       (operator name, theme, layout) — written when the
#                       dashboard starts the backup; a scheduled
#                       backup leaves the repo's last copy in place
#   chips/              installed.json + the chip .png files
#   rag-packs/          installed.json + each pack folder whole
#                       (documents, built index, metadata — the
#                       installs record carries the active states)
#   chips/workflows.json  her saved workflows
#   mesh-cards/         own.png — this node's pairing-card picture,
#                       minted fresh at backup time — plus one
#                       peer-<node>.png per paired peer, re-drawn
#                       from the stored card (the payload is the
#                       card, so a restored picture imports whole).
#                       The node keypair is never among them.
#   chats/              chats.json + uploads.json — the server-side
#                       chat store (Build 12, #139): the threads
#                       themselves and the uploads index. A restore
#                       CONVERGES them (newest-wins per thread, and
#                       diverged threads survive as dated siblings)
#                       rather than overwriting the store.
# Never backed up: call logs, caches and logs, the hardware
# profile, and voice model files (they reinstall).
# Uploads go through the Contents API; downloads for import go
# through git trees/blobs so a large chip restores whole.
# Scheduling is honest: a thread checks on launch and while the
# bridge runs whether a backup is due (last backup + the operator's
# interval of 1/3/5/7 days) and runs it when signed in — backups
# happen while Navi is running, not while the machine is off.

GITHUB_TOKEN_PATH = os.path.join(ROOT, "config", "github-auth.json")
GITHUB_REPO_URL = "https://github.com/AlexKollar/Apex"
GITHUB_API_BASE = "https://api.github.com/repos/AlexKollar/Apex"
GITHUB_BACKUP_ITEMS = ["soul", "memory", "settings", "chips", "rag", "workflows", "vault", "mesh-cards", "chats"]

# v0.5.1e (#256): the operator class inside a settings import,
# named deliberately. These keys are the operator's profile and
# standing settings, and they — only they — cross on the apply
# side of a backup import. Everything else in the config is
# machine or install state (github, bridge, hardware_profile,
# runtime, install_dir, mesh, dhd, public_server, startup,
# persona_files, backend, live, the version stamps) and never
# crosses: this machine keeps the facts its own install
# detected. The voice rides its own file, as it always has.
OPERATOR_SETTINGS_KEYS = ("operator_name", "threat_level", "sudo_mode",
                          "installed_commands", "updater")
# The always-ticked core (build .36): Soul, Memory, and Settings &
# config ride in every backup automatically — they are what makes
# an imported Navi *her*, so github_settings folds them into the
# stored list on every read and no save can drop them. Chips, RAG
# packs, and workflows stay the operator's choice.
GITHUB_CORE_BACKUP_ITEMS = ["soul", "memory", "settings"]
_github_device = {}

# GitHub sign-in client identity (0.2.0e Build 6, #173 phase 3):
# the OAuth App the operator signs in to is registered under the
# project owner's GitHub account with Device Flow enabled. Its
# client id is a PUBLIC identifier — no client secret exists or
# is needed for the device flow. The shipped default lives in
# config/github-signin.json (field client_id); an operator (or a
# fork) may override it there, or with the older github.client_id
# config value. Resolution order: the signin file, then config,
# then the shipped default constant below.
GITHUB_SIGNIN_PATH = os.path.join(ROOT, "config", "github-signin.json")
GITHUB_SIGNIN_DEFAULT_CLIENT_ID = "Ov23lijV9Q5ERi6nIAgG"


def github_signin_client_id(cfg=None):
    try:
        with open(GITHUB_SIGNIN_PATH) as f:
            data = json.load(f)
        if isinstance(data, dict) and str(data.get("client_id") or "").strip():
            return str(data["client_id"]).strip()
    except Exception:
        pass
    if cfg is not None:
        try:
            cid = github_settings(cfg)["client_id"]
            if cid:
                return cid
        except Exception:
            pass
    return GITHUB_SIGNIN_DEFAULT_CLIENT_ID


def github_read_token():
    # The one credential the app's GitHub reads ride (0.2.0e
    # Build 6): the OPERATOR's device-flow sign-in token. Build 2
    # provisioned a separate fine-grained PAT for app reads (the
    # "Apex Runtime" credential, config/github-runtime.json);
    # Build 6 supersedes it — no PAT to create, paste, or rotate,
    # and a leftover runtime file is ignored, never read. Signed
    # out, this returns None and the reads go unauthenticated
    # against the public API, exactly as they degrade in words.
    auth = github_auth()
    return auth["token"] if auth else None


def github_api(method, url, token=None, body=None, form=False):
    # Build 11 (#197): the shared authenticated-call wrapper.
    # When the call rides the operator's STORED token (the only
    # token any authenticated consumer passes), the token is
    # renewed BEFORE the call when its recorded expiry says it
    # is dead or nearly so, and a 401 that still comes back
    # earns exactly one renewal and one retry — one renewal
    # per call, never a loop. Calls carrying any other token
    # (the device flow's just-minted token, a token=None OAuth
    # exchange) pass straight through: renewal is a property
    # of the stored sign-in, not of the wire.
    stored = github_auth()
    if token and stored and token == stored.get("token"):
        fresh, _attempted = github_ensure_fresh_token()
        if fresh and fresh.get("token") and fresh["token"] != token:
            token = fresh["token"]
        status, resp = _github_api_once(method, url, token, body, form)
        if status == 401 and not _attempted:
            renewed_auth = github_refresh_auth()
            if (renewed_auth and renewed_auth.get("token")
                    and renewed_auth["token"] != token):
                status, resp = _github_api_once(
                    method, url, renewed_auth["token"], body, form)
        return status, resp
    return _github_api_once(method, url, token, body, form)


def _github_api_once(method, url, token=None, body=None, form=False):
    # One GitHub request; returns (status, parsed JSON or text) and
    # never raises for an HTTP error — the caller words the failure.
    headers = {
        "Accept": "application/vnd.github+json",
        "User-Agent": "navi-apex-bridge",
    }
    data = None
    if body is not None:
        if form:
            data = urllib.parse.urlencode(body).encode()
            headers["Content-Type"] = "application/x-www-form-urlencoded"
        else:
            data = json.dumps(body).encode()
            headers["Content-Type"] = "application/json"
    if token:
        headers["Authorization"] = "Bearer " + token
    req = urllib.request.Request(url, data=data, headers=headers, method=method)
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            _github_note_rate(resp.headers, url)
            raw = resp.read().decode("utf-8", "replace")
            try:
                return resp.status, json.loads(raw)
            except Exception:
                return resp.status, raw
    except urllib.error.HTTPError as e:
        _github_note_rate(e.headers, url)
        raw = e.read().decode("utf-8", "replace")
        try:
            return e.code, json.loads(raw)
        except Exception:
            return e.code, raw
    except Exception as e:
        return 0, str(e)


def github_api_raw(method, url, token, data, content_type):
    # Build 11 (#197): the raw sibling rides the same renewal
    # wrapper as github_api — stored token, pre-emptive renewal,
    # one renewal + one retry on a 401, never a loop.
    stored = github_auth()
    if token and stored and token == stored.get("token"):
        fresh, _attempted = github_ensure_fresh_token()
        if fresh and fresh.get("token") and fresh["token"] != token:
            token = fresh["token"]
        status, resp = _github_api_raw_once(method, url, token, data, content_type)
        if status == 401 and not _attempted:
            renewed_auth = github_refresh_auth()
            if (renewed_auth and renewed_auth.get("token")
                    and renewed_auth["token"] != token):
                status, resp = _github_api_raw_once(
                    method, url, renewed_auth["token"], data, content_type)
        return status, resp
    return _github_api_raw_once(method, url, token, data, content_type)


def _github_api_raw_once(method, url, token, data, content_type):
    # github_api's sibling for binary bodies (release assets ride
    # the uploads host as raw bytes, not JSON). Same contract:
    # (status, parsed JSON or text), never raises for HTTP errors.
    headers = {
        "Accept": "application/vnd.github+json",
        "User-Agent": "navi-apex-bridge",
        "Content-Type": content_type,
    }
    if token:
        headers["Authorization"] = "Bearer " + token
    req = urllib.request.Request(url, data=data, headers=headers, method=method)
    try:
        with urllib.request.urlopen(req, timeout=60) as resp:
            _github_note_rate(resp.headers, url)
            raw = resp.read().decode("utf-8", "replace")
            try:
                return resp.status, json.loads(raw)
            except Exception:
                return resp.status, raw
    except urllib.error.HTTPError as e:
        _github_note_rate(e.headers, url)
        raw = e.read().decode("utf-8", "replace")
        try:
            return e.code, json.loads(raw)
        except Exception:
            return e.code, raw
    except Exception as e:
        return 0, str(e)


def github_err_text(resp):
    if isinstance(resp, dict):
        return str(resp.get("message") or resp.get("error_description") or resp.get("error") or resp)[:240]
    return str(resp)[:240]


def github_settings(cfg):
    gh = cfg.get("github") if isinstance(cfg.get("github"), dict) else {}
    if isinstance(gh.get("backup_items"), list):
        items = [i for i in gh["backup_items"] if i in GITHUB_BACKUP_ITEMS]
    else:
        items = list(GITHUB_BACKUP_ITEMS)
    # The core trio is ticked automatically, whatever an older
    # config recorded: Soul, Memory, and Settings & config always
    # ride along, in the canonical order.
    items = [i for i in GITHUB_BACKUP_ITEMS if i in GITHUB_CORE_BACKUP_ITEMS or i in items]
    interval = gh.get("backup_interval_days")
    return {
        "client_id": str(gh.get("client_id") or "").strip(),
        "backup_repo": str(gh.get("backup_repo") or "navi-backup").strip() or "navi-backup",
        "backup_private": gh.get("backup_private") is not False,
        "backup_items": items,
        "backup_interval_days": interval if interval in (1, 3, 5, 7) else 1,
        "last_backup_at": gh.get("last_backup_at"),
        "last_backup_file_count": gh.get("last_backup_file_count"),
        "last_backup_result": str(gh.get("last_backup_result") or ""),
        "last_import_at": gh.get("last_import_at"),
        "last_import_result": str(gh.get("last_import_result") or ""),
    }


def save_github_settings(patch):
    try:
        with open(CONFIG_PATH) as f:
            cfg = json.load(f)
    except Exception:
        cfg = {}
    gh = cfg.get("github") if isinstance(cfg.get("github"), dict) else {}
    gh.update(patch)
    cfg["github"] = gh
    os.makedirs(os.path.dirname(CONFIG_PATH), exist_ok=True)
    with open(CONFIG_PATH, "w") as f:
        json.dump(cfg, f, indent=2)
    return github_settings(cfg)


def github_auth():
    try:
        with open(GITHUB_TOKEN_PATH) as f:
            data = json.load(f)
        if isinstance(data, dict) and data.get("token"):
            return {
                "token": str(data["token"]),
                "login": str(data.get("login") or ""),
                "refresh_token": str(data.get("refresh_token") or ""),
                "expires_in": data.get("expires_in"),
                "expires_at": data.get("expires_at"),
            }
    except Exception:
        pass
    return None


def github_save_auth(token, login, refresh_token=None, expires_in=None):
    # The whole token set GitHub returned. An expiring user token
    # also carries a refresh token and a lifetime; the first build
    # threw both away, so the day the access token expired every
    # authenticated call answered 401 forever, with no renewal
    # path (issue #14). A non-expiring sign-in returns neither and
    # stores exactly what it always stored.
    data = {"token": token, "login": login}
    if refresh_token:
        data["refresh_token"] = str(refresh_token)
    if expires_in:
        data["expires_in"] = expires_in
        # Build 11 (#197): the ABSOLUTE expiry, persisted beside
        # the lifetime — until this build nothing ever read
        # expires_in back, so no consumer could know the token
        # had died until GitHub's 401 said so, once per call,
        # forever. github_ensure_fresh_token reads this.
        try:
            data["expires_at"] = int(time.time()) + int(expires_in)
        except Exception:
            pass
    os.makedirs(os.path.dirname(GITHUB_TOKEN_PATH), exist_ok=True)
    fd = os.open(GITHUB_TOKEN_PATH, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    with os.fdopen(fd, "w") as f:
        json.dump(data, f)
    try:
        os.chmod(GITHUB_TOKEN_PATH, 0o600)
    except Exception:
        pass
    # v0.6.4e (#323): signing in is the moment the account
    # gate can judge the standing corpus — a seat that was
    # unsigned at upgrade gets its purge judgment now.
    try:
        sync_account_gate_sweep()
    except Exception:
        pass


def github_refresh_auth():
    # Renew an expired access token from the stored refresh token
    # — once. Returns the fresh auth dict (already saved), or None
    # when there is nothing to renew with or GitHub refuses; the
    # caller then asks the operator to sign in again.
    auth = github_auth()
    if not auth or not auth.get("refresh_token"):
        return None
    client_id = github_signin_client_id(load_config())
    if not client_id:
        return None
    st, resp = github_api(
        "POST", "https://github.com/login/oauth/access_token", None,
        {
            "client_id": client_id,
            "grant_type": "refresh_token",
            "refresh_token": auth["refresh_token"],
        },
        form=True,
    )
    if isinstance(resp, dict) and resp.get("access_token"):
        # GitHub rotates the refresh token on renewal: keep the
        # new one when it sends one, the stored one otherwise.
        github_save_auth(
            str(resp["access_token"]),
            auth["login"],
            resp.get("refresh_token") or auth["refresh_token"],
            resp.get("expires_in"),
        )
        return github_auth()
    return None


def github_ensure_fresh_token():
    # Build 11 (#197): pre-emptive renewal. When the stored
    # sign-in carries an absolute expiry (persisted by
    # github_save_auth since this build) and the token is
    # expired or inside the renewal margin, renew BEFORE the
    # first authenticated call instead of discovering the death
    # as a 401 on every surface, every morning. Returns
    # (auth, attempted): the auth dict to ride — the fresh one
    # when renewal worked, the stored one otherwise — and
    # whether a renewal was ATTEMPTED at all. The wrappers
    # spend a call's one renewal here or on a 401, never both:
    # a dead refresh token earns exactly one attempt per call,
    # not one per layer. When no expiry was ever stored
    # (sign-ins from before this build), nothing is guessed:
    # the 401 path in the wrappers covers it, and the first
    # successful renewal starts the clock for every later call.
    auth = github_auth()
    if not auth or not auth.get("refresh_token"):
        return auth, False
    exp = auth.get("expires_at")
    if exp is None:
        return auth, False
    try:
        exp = float(exp)
    except Exception:
        return auth, False
    if time.time() < exp - 300:
        return auth, False
    fresh = github_refresh_auth()
    return (fresh or auth), True


def github_status_payload(cfg):
    s = github_settings(cfg)
    auth = github_auth()
    client_id = github_signin_client_id(cfg)
    return {
        "ok": True,
        "repoUrl": GITHUB_REPO_URL,
        "issuesUrl": GITHUB_REPO_URL + "/issues",
        "clientId": client_id,
        "clientIdConfigured": bool(client_id),
        "signedIn": auth is not None,
        "login": auth["login"] if auth else "",
        "backupRepo": s["backup_repo"],
        "backupPrivate": s["backup_private"],
        "backupItems": s["backup_items"],
        "intervalDays": s["backup_interval_days"],
        "lastBackupAt": s["last_backup_at"],
        "lastBackupFileCount": s["last_backup_file_count"],
        "lastBackupResult": s["last_backup_result"],
        "lastImportAt": s["last_import_at"],
        "lastImportResult": s["last_import_result"],
        # The identities, side by side (Build 2, #173; Build 6
        # collapses them to one credential). Operator: the human's
        # device-flow sign-in — the ONLY GitHub credential Apex
        # holds. It acts as the operator for backup/import and
        # carries the app's reads (the Updater, the NUR, the
        # GitHub section) since Build 6 retired the separate
        # Runtime PAT: sign in once, with your account — no keys
        # to create, paste, or rotate. Release Automation:
        # deliberately NOT a credential Apex holds — publication
        # runs on the operator's signed-in browser session and the
        # local git remote, and the panel says so instead of
        # implying a second token exists. Bucket readouts are the
        # last limits GitHub actually reported to this bridge, or
        # null.
        "identities": {
            "operator": {
                "signedIn": auth is not None,
                "login": auth["login"] if auth else "",
                "scopesNote": "Acts as you — the only GitHub credential Apex holds. Used by backup and import, and by the app's own reads (Updater, NUR, the GitHub section).",
                "buckets": github_rate_state(),
            },
            "automation": {
                "note": "Releases are published from the operator's signed-in browser session and the local git remote. Apex holds no release credential — by design.",
            },
        },
    }


def github_issues_payload():
    # The Issues tab inside the GitHub section: the repo's current
    # open issues, live from the public GitHub API. The repo is
    # public and listing never requires sign-in; when the Runtime
    # credential is configured the read rides it (Build 2, #173),
    # so the section stops sharing the unauthenticated bucket
    # whose exhaustion presented as this tab's bare 403s (#172).
    # Pull requests ride the same endpoint and are filtered out.
    _rt = github_read_token()
    st, resp = github_api(
        "GET",
        GITHUB_API_BASE + "/issues?state=open&per_page=100&sort=updated&direction=desc",
        _rt,
    )
    if st != 200 or not isinstance(resp, list):
        _info = github_classify_error(st, None, github_err_text(resp), authed=bool(_rt))
        return {
            "ok": False,
            "error": _info["text"],
            "githubError": _info,
            "issues": [],
            "fetchedAt": None,
            "repoUrl": GITHUB_REPO_URL,
            "issuesUrl": GITHUB_REPO_URL + "/issues",
        }
    issues = []
    for i in resp:
        if not isinstance(i, dict) or "pull_request" in i:
            continue
        issues.append({
            "number": i.get("number") or 0,
            "title": str(i.get("title") or ""),
            "url": str(i.get("html_url") or ""),
            "state": str(i.get("state") or "open"),
            "labels": [str(l.get("name")) for l in (i.get("labels") or []) if isinstance(l, dict) and l.get("name")],
            "author": str((i.get("user") or {}).get("login") or ""),
            "updatedAt": str(i.get("updated_at") or ""),
            "createdAt": str(i.get("created_at") or ""),
            "comments": i.get("comments") or 0,
        })
    return {
        "ok": True,
        "error": None,
        "issues": issues,
        "fetchedAt": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "repoUrl": GITHUB_REPO_URL,
        "issuesUrl": GITHUB_REPO_URL + "/issues",
    }


def github_pulse_payload():
    # The GitHub section's Overview: the project's pulse — repo
    # description, stars/forks/watchers, latest release, recent
    # commits, open-issue count, last push — live from the public
    # GitHub API (on the operator's sign-in when one exists,
    # Build 6). Any failure is reported, never filled in with
    # invented numbers.
    st, repo = github_api("GET", GITHUB_API_BASE, github_read_token())
    if st != 200 or not isinstance(repo, dict):
        _info = github_classify_error(st, None, github_err_text(repo), authed=bool(github_read_token()))
        return {
            "ok": False,
            "error": _info["text"],
            "githubError": _info,
            "pulse": None,
            "fetchedAt": None,
            "repoUrl": GITHUB_REPO_URL,
        }
    latest_release = None
    rst, rel = github_api("GET", GITHUB_API_BASE + "/releases/latest", github_read_token())
    if rst == 200 and isinstance(rel, dict):
        latest_release = {
            "tag": str(rel.get("tag_name") or ""),
            "name": str(rel.get("name") or ""),
            "url": str(rel.get("html_url") or ""),
            "publishedAt": str(rel.get("published_at") or ""),
        }
    commits = []
    commit_count = None
    cst, crows = github_api("GET", GITHUB_API_BASE + "/commits?per_page=100", github_read_token())
    if cst == 200 and isinstance(crows, list):
        if len(crows) < 100:
            commit_count = len(crows)
        for c in crows[:5]:
            if not isinstance(c, dict):
                continue
            cm = c.get("commit") or {}
            au = cm.get("author") or {}
            commits.append({
                "sha": str(c.get("sha") or "")[:7],
                "message": str(cm.get("message") or "").split("\n")[0],
                "author": str(au.get("name") or ""),
                "date": str(au.get("date") or ""),
                "url": str(c.get("html_url") or ""),
            })
    issues = []
    ist, irows = github_api("GET", GITHUB_API_BASE + "/issues?state=open&per_page=8", github_read_token())
    if ist == 200 and isinstance(irows, list):
        for i in irows:
            if not isinstance(i, dict) or "pull_request" in i:
                continue
            issues.append({
                "number": i.get("number") or 0,
                "title": str(i.get("title") or ""),
                "url": str(i.get("html_url") or ""),
                "state": str(i.get("state") or "open"),
                "labels": [str(l.get("name")) for l in (i.get("labels") or []) if isinstance(l, dict) and l.get("name")],
                "author": str((i.get("user") or {}).get("login") or ""),
                "updatedAt": str(i.get("updated_at") or ""),
            })
            if len(issues) >= 8:
                break
    pulse = {
        "description": str(repo.get("description") or ""),
        "stars": repo.get("stargazers_count") or 0,
        "forks": repo.get("forks_count") or 0,
        "watchers": repo.get("subscribers_count") or repo.get("watchers_count") or 0,
        "language": str(repo.get("language") or "—"),
        "openIssuesCount": repo.get("open_issues_count") or 0,
        "pushedAt": str(repo.get("pushed_at") or ""),
        "defaultBranch": str(repo.get("default_branch") or "main"),
        "commitCount": commit_count,
        "latestRelease": latest_release,
        "commits": commits,
        "issues": issues,
    }
    return {
        "ok": True,
        "error": None,
        "pulse": pulse,
        "fetchedAt": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "repoUrl": GITHUB_REPO_URL,
    }


def github_device_start(cfg):
    client_id = github_signin_client_id(cfg)
    if not client_id:
        return {"ok": False, "error": "No GitHub OAuth client id is configured — sign-in can't start until one is set in config/github-signin.json. Viewing issues needs no sign-in at all."}
    st, resp = github_api(
        "POST", "https://github.com/login/device/code", None,
        {"client_id": client_id, "scope": "repo"}, form=True,
    )
    if st != 200 or not isinstance(resp, dict) or not resp.get("device_code"):
        return {"ok": False, "error": "GitHub did not start a device sign-in. " + github_classify_error(st, None, github_err_text(resp), authed=False)["text"]}
    _github_device.clear()
    _github_device.update({
        "device_code": resp["device_code"],
        "interval": resp.get("interval") or 5,
    })
    return {
        "ok": True,
        "userCode": resp.get("user_code") or "",
        "verificationUri": resp.get("verification_uri") or "https://github.com/login/device",
        "expiresIn": resp.get("expires_in"),
        "interval": resp.get("interval") or 5,
    }


def github_device_poll(cfg):
    s = github_settings(cfg)
    device_code = _github_device.get("device_code")
    if not device_code:
        return {"ok": False, "status": "idle", "error": "No sign-in is in progress — start one first."}
    st, resp = github_api(
        "POST", "https://github.com/login/oauth/access_token", None,
        {
            "client_id": github_signin_client_id(cfg),
            "device_code": device_code,
            "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
        },
        form=True,
    )
    if isinstance(resp, dict) and resp.get("access_token"):
        token = str(resp["access_token"])
        login = ""
        ust, user = github_api("GET", "https://api.github.com/user", token)
        if ust == 200 and isinstance(user, dict):
            login = str(user.get("login") or "")
        # Keep the whole token set: when GitHub answers with an
        # expiring token, the refresh token and lifetime ride
        # along — without them the sign-in dies with the access
        # token and cannot be renewed (issue #14).
        github_save_auth(token, login, resp.get("refresh_token"), resp.get("expires_in"))
        _github_device.clear()
        return {"ok": True, "status": "authorized", "login": login}
    err = resp.get("error") if isinstance(resp, dict) else None
    if err == "authorization_pending":
        return {"ok": True, "status": "pending"}
    if err == "slow_down":
        return {"ok": True, "status": "pending", "slowDown": True}
    _github_device.clear()
    if err:
        return {"ok": False, "status": "failed", "error": "GitHub sign-in ended (" + str(err) + ") — start again to retry."}
    return {"ok": False, "status": "failed", "error": "GitHub sign-in ended. " + github_classify_error(st, None, github_err_text(resp), authed=False)["text"] + " Start again to retry."}


def _dashboard_image_field(value):
    # Dashboard-carried image fields (operator picture / operator badge,
    # issue #116): only a data:image/ URL of sane size rides the backup
    # slice. Anything else drops to empty - never trusted, never fatal.
    if isinstance(value, str) and value.startswith("data:image/") and len(value) <= 420000:
        return value
    return ""


def _dashboard_image_bytes(field):
    # The raw bytes a dashboard image field carries (Build 6,
    # #180): the data URL's payload, decoded — or None when the
    # field is not a well-formed base64 data URL. Byte-exact by
    # construction; the Operator card's plain-file backup and its
    # byte-for-byte restore both stand on this.
    if not field or "," not in str(field):
        return None
    try:
        return base64.b64decode(str(field).split(",", 1)[1], validate=True)
    except Exception:
        return None


def github_collect_backup(cfg, dashboard=None):
    # The ready-to-fly essentials as {path: bytes}. Only the ticked
    # essentials are gathered; the manifest always rides along and
    # says exactly what this backup holds.
    s = github_settings(cfg)
    items = s["backup_items"]
    files = {}
    if "soul" in items:
        files["soul.md"] = read_file(SOUL_PATH).encode()
    if "memory" in items:
        files["memory.md"] = read_file(MEMORY_PATH).encode()
    if "settings" in items:
        clean = {
            k: v for k, v in cfg.items()
            if k not in ("github", "bridge", "hardware_profile", "runtime")
        }
        if isinstance(clean.get("intel"), dict):
            clean["intel"] = {
                k: v for k, v in clean["intel"].items()
                if k != "watchlist"
            }
        files["settings.json"] = json.dumps(clean, indent=2).encode()
        files["voice.json"] = json.dumps(cfg.get("voice") or {}, indent=2).encode()
        if isinstance(dashboard, dict):
            files["dashboard.json"] = json.dumps({
                "operatorName": str(dashboard.get("operatorName") or ""),
                "operatorPicture": _dashboard_image_field((dashboard or {}).get("operatorPicture")),
                "operatorBadge": _dashboard_image_field((dashboard or {}).get("operatorBadge")),
                "theme": str(dashboard.get("theme") or ""),
                "layout": str(dashboard.get("layout") or ""),
                "motion": str(dashboard.get("motion") or ""),
                "uiScale": (dashboard.get("uiScale")
                            if dashboard.get("uiScale") in (1, 1.1, 1.25, 1.5) else ""),
            }, indent=2).encode()
            # Build 6 (#180): the Operator card ALSO travels as
            # its own plain file — cards/operator-badge.png,
            # exactly the card's bytes, no archive — so it can be
            # listed and fetched one card at a time (on the phone,
            # over DHD) without opening the whole backup. The
            # dashboard.json copy above stays (older restores read
            # it); the card file holds the same bytes, openly.
            # The Mesh node keypair is never in any of this: it
            # is not read here, and nothing under mesh/ below
            # moves more than pairing-card pictures.
            _card_field = _dashboard_image_field((dashboard or {}).get("operatorBadge"))
            _card_bytes = _dashboard_image_bytes(_card_field) if _card_field else None
            if _card_bytes:
                files["cards/operator-badge.png"] = _card_bytes
    # The Mesh ledger rides a backup ONLY on the operator's
    # explicit tick (Settings → System → Navi Mesh; Q4, off by
    # default): it is metadata — who lent whom how many seconds —
    # but whose machines talked is the operator's business, so it
    # stays home unless they say otherwise. The node keypair never
    # rides a backup, tick or no tick.
    if mesh_settings(cfg)["ledgerInBackup"] and os.path.isfile(MESH_LEDGER_PATH):
        with open(MESH_LEDGER_PATH, "rb") as f:
            files["mesh/ledger.jsonl"] = f.read()
    # Mesh pairing cards (Build 9, #89): the pictures themselves,
    # on their own essential's tick. This node's own card is
    # minted fresh from the live card; each paired peer's picture
    # is re-drawn from its stored card (build_mesh_peer_card_png)
    # — the payload is the card, so a restored picture imports
    # whole. The node keypair is NOT a card and never rides: only
    # the public card data crosses, exactly what a pairing hands
    # to another machine in the open.
    if "mesh-cards" in items:
        if mesh_node_id() and mesh_cert_fingerprint():
            try:
                files["mesh-cards/own.png"] = build_mesh_card_png(cfg)
            except Exception:
                pass
        mesh_block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
        borrow_block = mesh_block.get("borrow") if isinstance(mesh_block.get("borrow"), dict) else {}
        for peer in borrow_block.get("peers") or []:
            if not isinstance(peer, dict):
                continue
            node_id = str(peer.get("nodeId") or "")
            if len(node_id) != 64:
                continue
            try:
                files["mesh-cards/peer-" + node_id[:12] + ".png"] = build_mesh_peer_card_png(peer)
            except Exception:
                continue
    if "chips" in items:
        files["chips/installed.json"] = (read_file(CHIP_INSTALLS_PATH) or "[]").encode()
        for row in load_chip_installs():
            local_file = row.get("local_file")
            if local_file:
                path = os.path.join(CHIPS_DIR, os.path.basename(str(local_file)))
                if os.path.isfile(path):
                    with open(path, "rb") as f:
                        files["chips/" + os.path.basename(str(local_file))] = f.read()
    if "rag" in items:
        files["rag-packs/installed.json"] = (read_file(RAG_INSTALLS_PATH) or "[]").encode()
        if os.path.isdir(RAG_PACKS_DIR):
            for slug in sorted(os.listdir(RAG_PACKS_DIR)):
                pdir = os.path.join(RAG_PACKS_DIR, slug)
                if not os.path.isdir(pdir):
                    continue
                for name in sorted(os.listdir(pdir)):
                    fpath = os.path.join(pdir, name)
                    if os.path.isfile(fpath):
                        with open(fpath, "rb") as f:
                            files["rag-packs/" + slug + "/" + name] = f.read()
    if "workflows" in items:
        files["chips/workflows.json"] = (read_file(WORKFLOWS_FILE) or "[]").encode()
    if "vault" in items and os.path.isdir(VAULT_DIR):
        for base, dirs, names in os.walk(VAULT_DIR):
            dirs.sort()
            for name in sorted(names):
                fpath = os.path.join(base, name)
                if os.path.isfile(fpath):
                    rel = os.path.relpath(fpath, VAULT_DIR).replace(os.sep, "/")
                    with open(fpath, "rb") as f:
                        files["vault/" + rel] = f.read()
    if "chats" in items:
        _chat_payload = chats_payload()
        files["chats/chats.json"] = json.dumps({"chats": _chat_payload["chats"], "exportedAt": _now_iso()}, indent=2).encode()
        files["chats/uploads.json"] = json.dumps(_chat_payload["uploads"], indent=2).encode()
    # The manifest's app name is the product name, "Apex" — it is
    # written into navi-backup.json in the operator's backup repo,
    # where it is read as identity, never matched as a token. (The
    # /api/status app value is a different thing: a bridge-recognition
    # token that stays "Navi: Apex" so bridges and launchers from
    # different builds keep recognizing each other; nothing displays
    # it, and restores validate only the format field below, never
    # this name — so backups written by older builds still restore.)
    manifest = {
        "format": "navi-backup/1",
        "app": "Apex",
        "bridgeBuild": BRIDGE_BUILD,
        "backedUpAt": _now_iso(),
        "items": items,
    }
    out = {"navi-backup.json": b""}
    out.update(files)
    # The manifest records the backup's file count — the files the
    # backup actually contains, exactly as Settings → GitHub & Backup
    # presents them: one per ticked essential's payload files. The
    # manifest itself (navi-backup.json) is the index, not content,
    # and is never counted — counting it (plus expanding settings
    # into three files) is what pinned the old readout at "6 files"
    # no matter what was ticked. The status column's simple
    # "# files | # days ago" readout and Settings → GitHub & Backup
    # both read this one record, so they can never disagree.
    manifest["fileCount"] = len(files)
    out["navi-backup.json"] = json.dumps(manifest, indent=2).encode()
    return out, manifest


def github_run_backup(cfg, dashboard=None):
    auth = github_auth()
    if not auth:
        return {"ok": False, "error": "Not signed in to GitHub — a backup writes to your GitHub account, so it needs the sign-in. Navi herself never does."}
    s = github_settings(cfg)
    if not s["backup_items"]:
        return {"ok": False, "error": "Nothing is ticked in the essentials checklist — an empty backup would restore an empty Navi."}
    login, token, repo = auth["login"], auth["token"], s["backup_repo"]
    base = "https://api.github.com/repos/" + login + "/" + repo
    st, _ = github_api("GET", base, token)
    if st == 404:
        st, created = github_api("POST", "https://api.github.com/user/repos", token, {
            "name": repo,
            "private": s["backup_private"],
            "description": "Apex backup — her ready-to-fly essentials (Soul, Memory, settings, chips, RAG packs, workflows, the Vault, Mesh pairing cards, chats).",
            "auto_init": True,
        })
        if st not in (200, 201):
            return {"ok": False, "error": "GitHub refused to create the backup repo " + login + "/" + repo + " (HTTP " + str(st) + "): " + github_err_text(created)}
    elif st != 200:
        return {"ok": False, "error": "Could not reach the backup repo " + login + "/" + repo + " (HTTP " + str(st) + ")."}
    files, manifest = github_collect_backup(cfg, dashboard)
    for path in sorted(files):
        url = base + "/contents/" + urllib.parse.quote(path, safe="/")
        sha = None
        gst, existing = github_api("GET", url, token)
        if gst == 200 and isinstance(existing, dict):
            sha = existing.get("sha")
        body = {
            "message": "Navi backup " + manifest["backedUpAt"],
            "content": base64.b64encode(files[path]).decode(),
        }
        if sha:
            body["sha"] = sha
        pst, resp = github_api("PUT", url, token, body)
        if pst not in (200, 201):
            return {"ok": False, "error": "GitHub refused " + path + " (HTTP " + str(pst) + "): " + github_err_text(resp)}
    result = (
        str(len(manifest["items"])) + " essentials written to " + login + "/" + repo
        + (" (private)" if s["backup_private"] else " (PUBLIC)")
    )
    save_github_settings({"last_backup_at": manifest["backedUpAt"], "last_backup_file_count": manifest.get("fileCount"), "last_backup_result": result})
    return {"ok": True, "result": result, "files": sorted(files), "backedUpAt": manifest["backedUpAt"]}


def github_fetch_backup(cfg):
    # Pulls the backup repo's files down for an import: returns
    # ({"files": {path: bytes}, "manifest": {...}}, None) or
    # (None, error). Downloads go through git trees/blobs so files
    # over the Contents API's size limit still restore whole.
    auth = github_auth()
    if not auth:
        return None, "Not signed in to GitHub — import reads your backup repo, so it needs the sign-in."
    s = github_settings(cfg)
    login, token, repo = auth["login"], auth["token"], s["backup_repo"]
    base = "https://api.github.com/repos/" + login + "/" + repo
    st, repo_j = github_api("GET", base, token)
    if st == 404:
        return None, "No backup repo yet — " + login + "/" + repo + " doesn't exist. Run a backup first."
    if st != 200 or not isinstance(repo_j, dict):
        return None, "Could not read " + login + "/" + repo + " (HTTP " + str(st) + ")."
    branch = str(repo_j.get("default_branch") or "main")
    st, tree = github_api("GET", base + "/git/trees/" + urllib.parse.quote(branch, safe="") + "?recursive=1", token)
    if st != 200 or not isinstance(tree, dict):
        return None, "Could not list the backup's files (HTTP " + str(st) + ")."
    blobs = {}
    for entry in tree.get("tree") or []:
        if entry.get("type") == "blob" and entry.get("path"):
            blobs[str(entry["path"])] = entry.get("sha")
    if "navi-backup.json" not in blobs:
        return None, login + "/" + repo + " doesn't hold a Navi backup (no navi-backup.json manifest)."
    fixed = {
        "navi-backup.json", "soul.md", "memory.md", "settings.json", "voice.json",
        "dashboard.json", "chips/installed.json", "chips/workflows.json",
        "rag-packs/installed.json", "chats/chats.json", "chats/uploads.json",
    }
    files = {}
    for path, sha in blobs.items():
        if path in fixed or path.startswith("rag-packs/") or path.startswith("vault/") or path.startswith("mesh-cards/") or path.startswith("cards/") or path.startswith("chats/") or (path.startswith("chips/") and path.endswith(".png")):
            bst, blob = github_api("GET", base + "/git/blobs/" + str(sha), token)
            if bst == 200 and isinstance(blob, dict) and blob.get("content"):
                try:
                    files[path] = base64.b64decode(str(blob["content"]))
                except Exception:
                    pass
    try:
        manifest = json.loads(files["navi-backup.json"].decode("utf-8", "replace"))
    except Exception:
        return None, "The backup's manifest could not be read."
    if not isinstance(manifest, dict) or manifest.get("format") != "navi-backup/1":
        return None, "That repo's manifest isn't a navi-backup/1 backup."
    return {"files": files, "manifest": manifest}, None


def github_import_changes(cfg, fetched, dashboard=None):
    files, manifest = fetched["files"], fetched["manifest"]
    items = [i for i in (manifest.get("items") or []) if i in GITHUB_BACKUP_ITEMS]
    labels = {
        "soul": "Soul document",
        "memory": "Memory document",
        "settings": "Settings & config",
        "chips": "Installed chips",
        "rag": "RAG packs",
        "workflows": "Workflows",
        "vault": "The Vault",
        "mesh-cards": "Mesh pairing cards",
        "chats": "Chats",
    }
    changes = []
    if "soul" in items:
        cur = read_file(SOUL_PATH)
        new = files.get("soul.md", b"").decode("utf-8", "replace")
        changes.append({
            "item": "soul", "label": labels["soul"],
            "detail": "Already matches the backup — nothing to change." if cur == new else "Would replace the current Soul (" + str(len(cur)) + " characters) with the backup's (" + str(len(new)) + " characters).",
            "willChange": cur != new,
        })
    if "memory" in items:
        cur = read_file(MEMORY_PATH)
        new = files.get("memory.md", b"").decode("utf-8", "replace")
        changes.append({
            "item": "memory", "label": labels["memory"],
            "detail": "Already matches the backup — nothing to change." if cur == new else "Would replace the current Memory (" + str(len(cur)) + " characters) with the backup's (" + str(len(new)) + " characters).",
            "willChange": cur != new,
        })
    if "settings" in items:
        diffs = []
        imported = {}
        try:
            imported = json.loads(files.get("settings.json", b"{}").decode("utf-8", "replace"))
        except Exception:
            imported = {}
        if isinstance(imported, dict):
            if "operator_name" in imported and str(imported.get("operator_name") or "") != str(cfg.get("operator_name") or ""):
                diffs.append("operator name in config")
            if "threat_level" in imported and imported.get("threat_level") != cfg.get("threat_level"):
                diffs.append("threat level")
            if "installed_commands" in imported and imported.get("installed_commands") != cfg.get("installed_commands"):
                diffs.append("installed commands")
        if "voice.json" in files:
            try:
                backed_voice = json.loads(files["voice.json"].decode("utf-8", "replace"))
            except Exception:
                backed_voice = None
            if isinstance(backed_voice, dict) and backed_voice != (cfg.get("voice") or {}):
                diffs.append("voice choice & settings")
        dash = {}
        try:
            dash = json.loads(files.get("dashboard.json", b"{}").decode("utf-8", "replace"))
        except Exception:
            dash = {}
        if isinstance(dash, dict) and dash and isinstance(dashboard, dict):
            if str(dash.get("operatorName") or "") != str(dashboard.get("operatorName") or ""):
                diffs.append("operator name → " + (str(dash.get("operatorName") or "") or "Operator"))
            if str(dash.get("theme") or "") != str(dashboard.get("theme") or ""):
                diffs.append("theme → " + str(dash.get("theme") or ""))
            if str(dash.get("layout") or "") != str(dashboard.get("layout") or ""):
                diffs.append("layout → " + str(dash.get("layout") or ""))
            if str(dash.get("motion") or "") != str(dashboard.get("motion") or ""):
                diffs.append("motion → " + str(dash.get("motion") or ""))
            if dash.get("uiScale") != dashboard.get("uiScale"):
                diffs.append("UI scale → " + str(dash.get("uiScale") or ""))
        changes.append({
            "item": "settings", "label": labels["settings"],
            "detail": ("Would restore: " + " · ".join(diffs) + ".") if diffs else "Already matches the backup — nothing to change.",
            "willChange": bool(diffs),
        })
    if "chips" in items:
        backed = []
        try:
            backed = json.loads(files.get("chips/installed.json", b"[]").decode("utf-8", "replace"))
        except Exception:
            backed = []
        cur = load_chip_installs()
        backed_names = sorted(str(r.get("full_name") or "") for r in backed if isinstance(r, dict)) if isinstance(backed, list) else []
        cur_names = sorted(str(r.get("full_name") or "") for r in cur if isinstance(r, dict))
        changes.append({
            "item": "chips", "label": labels["chips"],
            "detail": "Would restore " + str(len(backed) if isinstance(backed, list) else 0) + " installed chips from the backup, chips included — ready to run. This machine currently records " + str(len(cur)) + ".",
            "willChange": backed_names != cur_names,
        })
    if "rag" in items:
        backed = []
        try:
            backed = json.loads(files.get("rag-packs/installed.json", b"[]").decode("utf-8", "replace"))
        except Exception:
            backed = []
        cur = load_rag_installs()
        active = 0
        if isinstance(backed, list):
            active = sum(1 for r in backed if isinstance(r, dict) and r.get("active", True))
        backed_sig = sorted((str(r.get("full_name") or ""), bool(r.get("active", True))) for r in backed if isinstance(r, dict)) if isinstance(backed, list) else []
        cur_sig = sorted((str(r.get("full_name") or ""), bool(r.get("active", True))) for r in cur if isinstance(r, dict))
        changes.append({
            "item": "rag", "label": labels["rag"],
            "detail": "Would restore " + str(len(backed) if isinstance(backed, list) else 0) + " packs from the backup — documents, built indexes, and active states (" + str(active) + " active). This machine currently records " + str(len(cur)) + ".",
            "willChange": backed_sig != cur_sig,
        })
    if "workflows" in items:
        backed = []
        try:
            backed = json.loads(files.get("chips/workflows.json", b"[]").decode("utf-8", "replace"))
        except Exception:
            backed = []
        cur = load_workflows()
        backed_names = sorted(str(w.get("name") or "") for w in backed if isinstance(w, dict)) if isinstance(backed, list) else []
        cur_names = sorted(str(w.get("name") or "") for w in cur if isinstance(w, dict))
        changes.append({
            "item": "workflows", "label": labels["workflows"],
            "detail": "Would restore " + str(len(backed) if isinstance(backed, list) else 0) + " workflows from the backup. This machine currently holds " + str(len(cur)) + ".",
            "willChange": backed_names != cur_names,
        })
    if "vault" in items:
        backed_paths = sorted(p for p in files if p.startswith("vault/"))
        cur_paths = []
        if os.path.isdir(VAULT_DIR):
            for base, dirs, names in os.walk(VAULT_DIR):
                for name in names:
                    cur_paths.append("vault/" + os.path.relpath(os.path.join(base, name), VAULT_DIR).replace(os.sep, "/"))
        changes.append({
            "item": "vault", "label": labels["vault"],
            "detail": "Would restore " + str(len(backed_paths)) + " vault notes from the backup — the whole shared desk. This machine currently holds " + str(len(cur_paths)) + ".",
            "willChange": backed_paths != sorted(cur_paths),
        })
    if "mesh-cards" in items:
        backed_peers = sorted(p for p in files if p.startswith("mesh-cards/peer-") and p.endswith(".png"))
        backed_ids = sorted(os.path.basename(p)[5:-4] for p in backed_peers)
        mesh_block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
        borrow_block = mesh_block.get("borrow") if isinstance(mesh_block.get("borrow"), dict) else {}
        cur_ids = sorted(str(p.get("nodeId") or "")[:12] for p in (borrow_block.get("peers") or []) if isinstance(p, dict))
        own_backed = "mesh-cards/own.png" in files
        own_here = os.path.isfile(os.path.join(MESH_CARDS_DIR, "own.png"))
        changes.append({
            "item": "mesh-cards", "label": labels["mesh-cards"],
            "detail": "Would restore " + str(len(backed_peers)) + " peer pairing-card picture(s) to config/mesh/cards/ and re-pair those peers through the normal card check" + ("; the backup also holds this machine's own card picture, kept as a file — it never replaces the identity of the machine restoring" if own_backed else "") + ". This machine is currently paired with " + str(len(cur_ids)) + ".",
            "willChange": backed_ids != cur_ids or (own_backed and not own_here),
        })
    if "chats" in items:
        backed_chats = []
        backed_uploads = []
        try:
            _parsed_chats = json.loads(files.get("chats/chats.json", b"{}").decode("utf-8", "replace"))
            if isinstance(_parsed_chats, dict) and isinstance(_parsed_chats.get("chats"), list):
                backed_chats = _parsed_chats["chats"]
        except Exception:
            backed_chats = []
        try:
            _parsed_up = json.loads(files.get("chats/uploads.json", b"[]").decode("utf-8", "replace"))
            if isinstance(_parsed_up, list):
                backed_uploads = _parsed_up
        except Exception:
            backed_uploads = []
        _prev = chats_converge_preview(backed_chats)
        _moving = _prev["added"] + _prev["replaced"] + _prev["siblings"]
        if not _moving and not backed_uploads:
            _detail = "Already matches the backup — nothing to change."
        elif not _moving:
            _detail = "The threads already match the backup — nothing to change."
        else:
            _detail = ("Would converge " + str(len(backed_chats)) + " backed-up thread(s) into this machine's chat store: "
                       + str(_prev["added"]) + " new, " + str(_prev["replaced"]) + " updated (newest wins), "
                       + str(_prev["siblings"]) + " diverged — kept whole as dated sibling threads, never merged away. "
                       "This machine currently holds " + str(len(chats_all())) + ".")
        changes.append({
            "item": "chats", "label": labels["chats"],
            "detail": _detail,
            "willChange": bool(_moving),
        })
    if "dashboard.json" in files:
        try:
            _backed_dash = json.loads(files["dashboard.json"].decode("utf-8", "replace"))
        except Exception:
            _backed_dash = None
        if isinstance(_backed_dash, dict):
            _cur_dash = dashboard if isinstance(dashboard, dict) else {}
            for _key, _label in (("operatorPicture", "Operator picture"), ("operatorBadge", "Operator badge")):
                _new = _dashboard_image_field(_backed_dash.get(_key))
                _cur = _dashboard_image_field(_cur_dash.get(_key))
                if _new and _new != _cur:
                    # Build 6 (#180): when the card's own file
                    # rides the backup, the Operator card item
                    # below speaks for the badge — one restore,
                    # one line.
                    if _key == "operatorBadge" and "cards/operator-badge.png" in files:
                        continue
                    changes.append({
                        "item": "settings", "label": _label,
                        "detail": "Would restore your " + _label.lower() + " from the backup.",
                        "willChange": True,
                    })
    if "cards/operator-badge.png" in files:
        # Build 6 (#180): the Operator card as its own restorable
        # thing — compared as raw bytes against the card this
        # machine currently holds, because byte-for-byte is the
        # promise the plain file makes.
        _card = files["cards/operator-badge.png"]
        _cur_dash = dashboard if isinstance(dashboard, dict) else {}
        _curf = _dashboard_image_field(_cur_dash.get("operatorBadge"))
        _curb = _dashboard_image_bytes(_curf) if _curf else None
        if _curb != _card:
            changes.append({
                "item": "settings", "label": "Operator card",
                "detail": "Would restore your Operator card from its own plain file in the backup (cards/operator-badge.png) — the card's exact bytes, fetched on its own, no archive to open.",
                "willChange": True,
            })
    return changes, items


def _github_safe_part(part):
    return bool(re.match(r"^[A-Za-z0-9._-]+$", part)) and part not in (".", "..")


def github_apply_import(cfg, fetched):
    files, manifest = fetched["files"], fetched["manifest"]
    items = [i for i in (manifest.get("items") or []) if i in GITHUB_BACKUP_ITEMS]
    applied = []
    if "soul" in items and "soul.md" in files:
        os.makedirs(os.path.dirname(SOUL_PATH), exist_ok=True)
        with open(SOUL_PATH, "wb") as f:
            f.write(files["soul.md"])
        applied.append("Soul document")
    if "memory" in items and "memory.md" in files:
        os.makedirs(os.path.dirname(MEMORY_PATH), exist_ok=True)
        with open(MEMORY_PATH, "wb") as f:
            f.write(files["memory.md"])
        applied.append("Memory document")
    if "settings" in items and "settings.json" in files:
        try:
            imported = json.loads(files["settings.json"].decode("utf-8", "replace"))
        except Exception:
            imported = None
        if isinstance(imported, dict):
            new_cfg = load_config()
            # Only the operator's own settings cross over — never the
            # github section, the bridge block, the install dir, or
            # (above all) the hardware profile / runtime flags: this
            # machine keeps the profile its own install detected.
            for key in OPERATOR_SETTINGS_KEYS:
                if key == "updater":
                    if isinstance(imported.get("updater"), dict):
                        new_cfg["updater"] = imported["updater"]
                    continue
                if key in imported:
                    new_cfg[key] = imported[key]
            if "voice.json" in files:
                try:
                    voice = json.loads(files["voice.json"].decode("utf-8", "replace"))
                    if isinstance(voice, dict) and voice:
                        new_cfg["voice"] = voice
                except Exception:
                    pass
            with open(CONFIG_PATH, "w") as f:
                json.dump(new_cfg, f, indent=2)
            applied.append("Settings & config")
    if "chips" in items:
        wrote = False
        for path, data in files.items():
            if path == "chips/installed.json":
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(CHIP_INSTALLS_PATH, "wb") as f:
                    f.write(data)
                wrote = True
            elif path.startswith("chips/") and path.endswith(".png") and _github_safe_part(os.path.basename(path)):
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, os.path.basename(path)), "wb") as f:
                    f.write(data)
                wrote = True
        if wrote:
            applied.append("Installed chips")
    if "rag" in items:
        wrote = False
        for path, data in files.items():
            if path == "rag-packs/installed.json":
                os.makedirs(RAG_PACKS_DIR, exist_ok=True)
                with open(RAG_INSTALLS_PATH, "wb") as f:
                    f.write(data)
                wrote = True
            elif path.startswith("rag-packs/"):
                parts = path.split("/")
                if len(parts) == 3 and _github_safe_part(parts[1]) and _github_safe_part(parts[2]):
                    pdir = os.path.join(RAG_PACKS_DIR, parts[1])
                    os.makedirs(pdir, exist_ok=True)
                    with open(os.path.join(pdir, parts[2]), "wb") as f:
                        f.write(data)
                    wrote = True
        if wrote:
            applied.append("RAG packs")
    if "workflows" in items and "chips/workflows.json" in files:
        os.makedirs(CHIPS_DIR, exist_ok=True)
        with open(WORKFLOWS_FILE, "wb") as f:
            f.write(files["chips/workflows.json"])
        applied.append("Workflows")
    if "vault" in items:
        wrote = False
        for path, data in files.items():
            if path.startswith("vault/"):
                rel = vault_join(path[len("vault/"):])
                if rel:
                    full = os.path.join(VAULT_DIR, *rel.split("/"))
                    os.makedirs(os.path.dirname(full), exist_ok=True)
                    with open(full, "wb") as f:
                        f.write(data)
                    wrote = True
        if wrote:
            applied.append("The Vault")
    if "chats" in items and "chats/chats.json" in files:
        try:
            _parsed_chats = json.loads(files["chats/chats.json"].decode("utf-8", "replace"))
            _backed_chats = _parsed_chats.get("chats") if isinstance(_parsed_chats, dict) else []
        except Exception:
            _backed_chats = []
        _backed_uploads = None
        if "chats/uploads.json" in files:
            try:
                _parsed_up = json.loads(files["chats/uploads.json"].decode("utf-8", "replace"))
                if isinstance(_parsed_up, list):
                    _backed_uploads = _parsed_up
            except Exception:
                _backed_uploads = None
        if isinstance(_backed_chats, list) and _backed_chats:
            _conv = chats_converge(_backed_chats, _backed_uploads)
            if _conv.get("changed"):
                applied.append("Chats")
    if "mesh-cards" in items:
        wrote = False
        fresh_cfg = load_config()
        for path in sorted(files):
            if not path.startswith("mesh-cards/") or not path.endswith(".png"):
                continue
            base = os.path.basename(path)
            if not _github_safe_part(base):
                continue
            os.makedirs(MESH_CARDS_DIR, exist_ok=True)
            with open(os.path.join(MESH_CARDS_DIR, base), "wb") as f:
                f.write(files[path])
            wrote = True
            if base.startswith("peer-"):
                # Peers come back through the SAME validation a
                # hand import passes — picture parsed, card
                # checked, record added by mesh_add_peer. A
                # picture that fails the check stays a file in the
                # cards folder and pairs nothing. This machine's
                # own card picture (own.png) is never imported: it
                # describes the machine the backup came FROM, and
                # restoring must not overwrite a live identity.
                peer, _perr = parse_mesh_card_png(files[path])
                if peer is not None:
                    mesh_add_peer(fresh_cfg, peer)
        if wrote:
            applied.append("Mesh pairing cards")
    dashboard_out = None
    if "dashboard.json" in files:
        try:
            parsed = json.loads(files["dashboard.json"].decode("utf-8", "replace"))
            if isinstance(parsed, dict):
                dashboard_out = {
                    "operatorName": str(parsed.get("operatorName") or ""),
                    "theme": str(parsed.get("theme") or ""),
                    "layout": (str(parsed.get("layout") or "")
                               if str(parsed.get("layout") or "") in ("standard", "lcars") else ""),
                    "motion": (str(parsed.get("motion") or "")
                               if str(parsed.get("motion") or "") in ("full", "potato") else ""),
                    "uiScale": (parsed.get("uiScale")
                                if parsed.get("uiScale") in (1, 1.1, 1.25, 1.5) else ""),
                    "operatorPicture": _dashboard_image_field(parsed.get("operatorPicture")),
                    "operatorBadge": _dashboard_image_field(parsed.get("operatorBadge")),
                }
        except Exception:
            dashboard_out = None
    if dashboard_out is not None and "cards/operator-badge.png" in files:
        # Build 6 (#180): on restore the card's own file is
        # canonical — its exact bytes, re-wrapped as the
        # dashboard's badge field. (dashboard.json's copy is the
        # same bytes in every backup this product writes; the
        # plain file wins because it is the one a phone can fetch
        # alone, over DHD, without the rest of the backup.)
        _cf = _dashboard_image_field(
            "data:image/png;base64,"
            + base64.b64encode(files["cards/operator-badge.png"]).decode("ascii"))
        if _cf:
            dashboard_out["operatorBadge"] = _cf
    taken = str(manifest.get("backedUpAt") or "")
    result = (
        "Imported the backup from " + (taken[:10] if taken else "an unknown date")
        + " — restored: " + (", ".join(applied) if applied else "nothing (the backup held no restorable sections)") + "."
    )
    save_github_settings({"last_import_at": _now_iso(), "last_import_result": result})
    return {"ok": True, "result": result, "applied": applied, "dashboard": dashboard_out}


def github_backup_due(cfg):
    s = github_settings(cfg)
    if not github_auth() or not s["backup_items"]:
        return False
    last = s["last_backup_at"]
    if not last:
        return True
    try:
        epoch = calendar.timegm(time.strptime(str(last)[:19], "%Y-%m-%dT%H:%M:%S"))
    except Exception:
        return True
    return (time.time() - epoch) >= s["backup_interval_days"] * 86400


def github_auto_backup_loop():
    # Interval backups, honestly scoped: the check runs shortly
    # after the bridge starts and again while it runs. A backup that
    # comes due while the machine is off runs at the next launch —
    # nothing here pretends otherwise.
    time.sleep(25)
    while True:
        try:
            cfg = load_config()
            if github_backup_due(cfg):
                github_run_backup(cfg)
        except Exception:
            pass
        time.sleep(1800)


def load_chip_installs():
    try:
        with open(CHIP_INSTALLS_PATH) as f:
            rows = json.load(f)
        return rows if isinstance(rows, list) else []
    except Exception:
        return []


def save_chip_installs(rows):
    os.makedirs(CHIPS_DIR, exist_ok=True)
    with open(CHIP_INSTALLS_PATH, "w") as f:
        json.dump(rows, f, indent=2)


def fetch_repo_chip(full_name):
    # One catalog entry for a repo: its GitHub metadata plus its
    # latest release (None when the repo has never cut one).
    repo = github_json("https://api.github.com/repos/" + full_name)
    latest = None
    try:
        rel = github_json(
            "https://api.github.com/repos/" + full_name + "/releases/latest"
        )
        latest = {
            "tag": rel.get("tag_name") or "",
            "name": rel.get("name") or "",
            "url": rel.get("html_url") or "",
            "publishedAt": rel.get("published_at") or "",
        }
    except (urllib.error.HTTPError, GithubHttpError) as e:
        # 404 is the only answer that means "no releases yet" — a
        # rate limit or an outage must surface as a failed catalog,
        # never as a chip that looks release-less.
        if e.code != 404:
            raise
        latest = None
    return {
        "fullName": repo.get("full_name") or full_name,
        "name": repo.get("name") or "",
        "owner": (repo.get("owner") or {}).get("login") or "",
        "description": repo.get("description") or "",
        "htmlUrl": repo.get("html_url") or "",
        "stars": repo.get("stargazers_count") or 0,
        "forks": repo.get("forks_count") or 0,
        "language": repo.get("language") or "—",
        "topics": repo.get("topics") or [],
        "updatedAt": repo.get("updated_at") or "",
        "pushedAt": repo.get("pushed_at") or "",
        "defaultBranch": repo.get("default_branch") or "main",
        "openIssues": repo.get("open_issues_count") or 0,
        "hasWiki": bool(repo.get("has_wiki")),
        "latestRelease": latest,
    }


def chip_install_record(repo):
    latest = repo["latestRelease"]
    return {
        "full_name": repo["fullName"],
        "name": repo["name"],
        "html_url": repo["htmlUrl"],
        "installed_tag": latest["tag"] if latest else None,
        "installed_ref": ("release " + latest["tag"]) if latest
        else (repo["defaultBranch"] + " (no releases yet)"),
        "installed_pushed_at": repo["pushedAt"] or None,
        "installed_at": _now_iso(),
    }


def chip_slug(name):
    return re.sub(r"[^a-z0-9]+", "-", (name or "").lower()).strip("-") or "chip"


def chip_cartridge_filename(repo, rows):
    # The chip a GitHub install writes into chips/, named for
    # the repo like a Studio chip is named for its chip. When
    # another install already owns that filename (a forged chip of
    # the same name), the owner's name joins the slug instead of
    # overwriting their chip.
    filename = chip_slug(repo["name"]) + ".png"
    taken = {
        r.get("local_file") for r in rows
        if r.get("full_name") != repo["fullName"]
    }
    if filename in taken:
        filename = chip_slug(repo["owner"] + "-" + repo["name"]) + ".png"
    return filename


def github_download(url):
    # Build 11 (#197): downloads ride the renewal wrapper too —
    # pre-emptive renewal when the stored expiry says so, and
    # one renewal + one retry on a 401 (an expired token used
    # to fail the Updater's asset download outright, every
    # morning, until the operator signed in again by hand).
    _auth, _attempted = github_ensure_fresh_token()
    try:
        return _github_download_once(url)
    except urllib.error.HTTPError as e:
        if e.code == 401 and not _attempted and github_refresh_auth():
            return _github_download_once(url)
        raise


def _github_download_once(url):
    # A runtime read like any other (Build 2): the operator's
    # sign-in token rides along. (Release-asset downloads
    # themselves are unmetered by GitHub; the header costs
    # nothing and keeps the read on the app's own identity.)
    headers, _authed = _github_read_headers()
    req = urllib.request.Request(url, headers=headers)
    with urllib.request.urlopen(req, timeout=30) as resp:
        return resp.read()


def fetch_chip_payload(repo):
    # Downloads the chip itself from its repo and returns
    # (payload, source_file_name). A navi-chips repo carries its
    # chip as a Python file in the repo root — the published chips
    # all do (cryptex.py in navi-cryptex, nmap.py in navi-nmap) —
    # so the source comes from the installed ref (the latest
    # release's tag when the repo cuts releases, otherwise the
    # default branch) and is forged into a chip payload
    # exactly as Chip Studio forges a local chip: from that moment
    # the installed chip IS a chip in chips/, indistinguishable
    # from a forged one — pill row, /chip, workflows, execution.
    # Raises ValueError with the reason when the repo holds nothing
    # runnable, so an install never records a chip that is not
    # actually on this machine.
    full = repo["fullName"]
    latest = repo["latestRelease"]
    ref = latest["tag"] if latest else (repo["defaultBranch"] or "main")
    # Build .86 (issue #79): the walk is RECURSIVE — a legacy chip
    # keeps its code in a package directory (app_cryptex/ and kin),
    # and a root-only listing never sees it, which is how a chip
    # could "install" and never run. Every file in the tree is
    # listed with its repo-relative path; package directories keep
    # their paths in the payload file set.
    entries = []

    def _walk(dir_path):
        url = (
            "https://api.github.com/repos/" + full + "/contents"
            + ("/" + dir_path if dir_path else "")
            + "?ref=" + urllib.parse.quote(ref, safe="")
        )
        listing = github_json(url)
        if not isinstance(listing, list):
            raise ValueError("GitHub did not return the repo's file list.")
        for item in listing:
            p = str(item.get("path") or "")
            if item.get("type") == "dir":
                _walk(p)
            elif item.get("type") == "file":
                entries.append((p, item))

    _walk("")
    pys = [(p, i) for (p, i) in entries if p.lower().endswith(".py")]
    if not pys:
        raise ValueError(
            "the repo has no Python chip file at " + ref
            + " — a navi-chips repo carries its chip as a .py file."
        )
    # A navi-chip.json manifest at the repo root (written by the
    # GitHub-ready export) names the entry file outright and
    # carries the chip's description and declared output.
    manifest = {}
    man_item = next(
        (i for (p, i) in entries if p == "navi-chip.json"),
        None,
    )
    if man_item is not None:
        try:
            parsed = json.loads(
                github_download(man_item["download_url"]).decode("utf-8", "replace")
            )
            if isinstance(parsed, dict):
                manifest = parsed
        except Exception:
            manifest = {}
    base = repo["name"].lower()
    stem = base[5:] if base.startswith("navi-") else base
    wants = []
    if manifest.get("entry"):
        wants.append(str(manifest["entry"]).lower())
    wants += ["root.py", stem + ".py", base + ".py", "chip.py", "main.py"]
    pick = None
    pick_path = ""
    for want in wants:
        for (p, i) in pys:
            if p.lower() == want or p.lower().rsplit("/", 1)[-1] == want:
                pick = i
                pick_path = p
                break
        if pick:
            break
    if pick is None and len(pys) == 1:
        pick_path, pick = pys[0]
    if pick is None:
        names = ", ".join(p for (p, _i) in pys)
        raise ValueError(
            "the repo holds several Python files (" + names
            + ") and none is the chip's entry — name the entry file "
            "root.py, or add a navi-chip.json manifest naming it."
        )
    # Every Python file in the tree comes down at its relative
    # path: the entry becomes the chip's code and the rest ride
    # along as its importable modules and packages, so a
    # multi-file or package chip runs exactly as authored.
    files = []
    code = ""
    for (p, item) in pys:
        text = github_download(item["download_url"]).decode("utf-8", "replace")
        if item is pick:
            code = text
        else:
            files.append({"path": p, "content": text[:200000]})
    if not code.strip():
        raise ValueError(pick_path + " is empty.")
    # Declared dependencies ride along as data (issue #79): a
    # chip-requirements.txt at the repo root is read and its lines
    # are stored on the payload — surfaced on the chip's card,
    # never installed automatically.
    requirements = []
    req_item = next((i for (p, i) in entries if p == "chip-requirements.txt"), None)
    if req_item is not None:
        try:
            req_text = github_download(req_item["download_url"]).decode("utf-8", "replace")
            requirements = [
                ln.strip() for ln in req_text.splitlines()
                if ln.strip() and not ln.strip().startswith("#")
            ][:50]
        except Exception:
            requirements = []
    payload = {
        "name": repo["name"],
        "description": str(manifest.get("description") or repo["description"] or ""),
        "code": code[:200000],
        "output": str(manifest.get("output") or ""),
        "requirements": requirements,
        "glyph": str(manifest.get("glyph") or ""),
        "icon": _manifest_icon(manifest),
        # Issue #99: the manifest's version rides inside the
        # picture, the way its glyph already does.
        "version": str(manifest.get("version") or ""),
        "files": _valid_chip_files(
            [{"path": "root.py", "content": code[:200000]}] + files,
            code[:200000],
        ),
    }
    return payload, pick_path


def _display_ref(ref):
    # Install records written before the chip terminology sweep can
    # carry a "cartridge …" ref (e.g. "cartridge testchip.png"); the
    # card badge reads the ref verbatim, so it is normalized to
    # "chip …" on the way out. The stored record is the operator's
    # and is never rewritten here.
    if ref and ref.startswith("cartridge "):
        return "chip " + ref[len("cartridge "):]
    return ref


def chip_catalog_view():
    # A failed search yields an error and just the local installs; a
    # search that succeeds but rate-limits some repos yields those
    # repos plus a warning — a chip is never silently dropped.
    error = None
    warning = None
    repos = []
    try:
        # Build 2 (#173): the search rides the Runtime credential
        # when configured, is cached for five minutes, and runs
        # one at a time (github_search_repositories); a failure is
        # classified and named, never a bare status.
        data = github_search_repositories("topic:navi-chips")
        failures = 0
        last_err = None
        for item in data.get("items") or []:
            full = item.get("full_name")
            if not full:
                continue
            try:
                repos.append(fetch_repo_chip(full))
            except Exception as fe:
                failures += 1
                last_err = fe
        if failures:
            warning = (
                "GitHub could not be fully read for "
                + str(failures)
                + " of the chips — the list may be incomplete. "
                + github_error_text(last_err, authed=bool(github_read_token()))
            )
    except Exception as e:
        error = "GitHub search failed. " + github_error_text(e, authed=bool(github_read_token()))
    installs = {row.get("full_name"): row for row in load_chip_installs()}
    # Issue #22: an entry also reads as installed when a chip with
    # the same slug is on this machine under another name — a
    # Studio-forged chip (its local/ record), the same repo
    # installed from a different owner, or a .png dropped into
    # chips/ by hand (no record at all). An exact full-name match
    # keeps the update math; a slug match claims only "installed"
    # — no update is offered against a lineage this machine did
    # not download from this repo.
    installs_ci = {}
    by_slug = {}
    for row in load_chip_installs():
        fn = row.get("full_name") or ""
        if fn:
            installs_ci.setdefault(fn.lower(), row)
        s = fn.split("/")[-1].lower()
        if s and s not in by_slug:
            by_slug[s] = row
    local_files = {}
    for lc in local_chips():
        local_files.setdefault(str(lc.get("slug") or "").lower(), lc.get("file"))
    chips = []
    for repo in repos:
        inst = installs.get(repo["fullName"]) or installs_ci.get(repo["fullName"].lower())
        slug_row = None
        slug_file = None
        if not inst:
            s = repo["fullName"].split("/")[-1].lower()
            slug_row = by_slug.get(s)
            slug_file = local_files.get(s)
        matched = inst or slug_row
        slug_matched = bool(not inst and (slug_row is not None or slug_file is not None))
        update = False
        if inst:
            if repo["latestRelease"]:
                update = inst.get("installed_tag") != repo["latestRelease"]["tag"]
            else:
                update = bool(
                    inst.get("installed_pushed_at")
                    and repo["pushedAt"] > inst["installed_pushed_at"]
                )
        if inst:
            ref = _display_ref(inst.get("installed_ref"))
        elif slug_row and str(slug_row.get("full_name") or "").startswith("local/"):
            ref = _display_ref(slug_row.get("installed_ref"))
        else:
            ref = None
        entry = dict(repo)
        entry["inCatalog"] = True
        entry["installed"] = bool(inst) or slug_matched
        entry["slugMatched"] = slug_matched
        entry["installedTag"] = matched.get("installed_tag") if matched else None
        entry["installedRef"] = ref
        entry["installedAt"] = matched.get("installed_at") if matched else None
        entry["updateAvailable"] = update
        entry["localFile"] = (matched.get("local_file") if matched else None) or slug_file
        entry["locallyEdited"] = bool(matched.get("locally_edited")) if matched else False
        chips.append(entry)
    seen = {c["fullName"] for c in chips}
    for full, inst in installs.items():
        if full in seen:
            continue
        chips.append({
            "fullName": full,
            "name": inst.get("name") or full.split("/")[-1],
            "owner": full.split("/")[0],
            "description": "",
            "htmlUrl": inst.get("html_url") or "",
            "stars": 0, "forks": 0, "language": "—", "topics": [],
            "updatedAt": "",
            "pushedAt": inst.get("installed_pushed_at") or "",
            "defaultBranch": "main", "openIssues": 0,
            "latestRelease": None,
            "inCatalog": False, "installed": True, "slugMatched": False,
            "installedTag": inst.get("installed_tag"),
            "installedRef": _display_ref(inst.get("installed_ref")),
            "installedAt": inst.get("installed_at"),
            "updateAvailable": False,
            "localFile": inst.get("local_file"),
            "locallyEdited": bool(inst.get("locally_edited")),
        })
    view = {"fetchedAt": _now_iso(), "chips": chips}
    if error:
        view["error"] = error
    if warning:
        view["warning"] = warning
    return view


def _png_chunk(kind, data):
    out = struct.pack(">I", len(data)) + kind + data
    out += struct.pack(">I", binascii.crc32(kind + data) & 0xFFFFFFFF)
    return out


# --- Cartridge faces (0.1.9e, issue #92 graphics pass) --------------
# A chip's face is DRAWN, not rolled: one chunky pixel glyph that
# means the chip's function, on the family shell — black body, coral
# notched bands top and bottom. The glyph is named in the chip's
# metadata ("glyph" in the payload, and in navi-chip.json at export):
# an explicit glyph wins; otherwise a keyword heuristic over the
# chip's name and description picks one; a chip nothing claims keeps
# the legacy name-seeded pattern face, which remains only as that
# fallback. Faces are minted at forge time — installed and published
# pictures are never rewritten.
#
# Build 3 (issue #101): the glyphs are redrawn as SOLID silhouettes
# on a 20x20 grid — the same grid as the QR-style tile field that now
# textures the window behind them — drawn as objects, not outlines:
# filled masses with a few negative-space cuts (a lock's keyhole, a
# die's pips, a terminal's prompt notch). "#" is the accent fill;
# "o" is the shade fill (a darker tone of the same accent), used as
# the interior tone on the heaviest masses so they read as solid
# without going flat. An author can also draw their own: a manifest
# "icon" in this same 20x20, three-character language outranks the
# built-in set (see valid_icon_art).

GLYPH_PIXELS = {
    "key": [
        "....................",
        "....######..........",
        "..##########........",
        ".############.......",
        ".####....####.......",
        "###........###......",
        "###........###......",
        "###........###......",
        ".####....####.......",
        ".############.......",
        "..##########........",
        "....#######.........",
        ".......####.........",
        ".......####.........",
        ".......####..####...",
        ".......####.........",
        ".......####..####...",
        ".......####.........",
        ".......####.........",
        "....................",
    ],
    "lock": [
        "....................",
        ".......######.......",
        ".....##########.....",
        "....###......###....",
        "....##........##....",
        "....##........##....",
        "....##........##....",
        "...##############...",
        "..################..",
        "..################..",
        "..#######..#######..",
        "..######....######..",
        "..######....######..",
        "..#######..#######..",
        "..################..",
        "..################..",
        "..################..",
        "...##############...",
        "....................",
        "....................",
    ],
    "hash": [
        "....................",
        "....##......##......",
        "....##......##......",
        "....##......##......",
        ".##################.",
        ".##################.",
        "....##......##......",
        "....##......##......",
        "....##......##......",
        "....##......##......",
        ".##################.",
        ".##################.",
        "....##......##......",
        "....##......##......",
        "....##......##......",
        "....##......##......",
        "....................",
        "....................",
        "....................",
        "....................",
    ],
    "text": [
        "....................",
        ".##################.",
        ".##################.",
        "....##..##..##......",
        "....#####.#####.....",
        "....##..##..##......",
        "....##..##..##......",
        "....#####.#####.....",
        "....##..##..##......",
        "....##..##..##......",
        "....#####.#####.....",
        "....##..##..##......",
        "....##..##..##......",
        "....#####.#####.....",
        "....##..##..##......",
        "....................",
        "....................",
        "....................",
        "....................",
        "....................",
    ],
    "swap": [
        "....................",
        "..........####......",
        "..........#####.....",
        "...################.",
        ".##################.",
        "###################.",
        "...################.",
        "..........#####.....",
        "..........####......",
        "....................",
        "....................",
        "......####..........",
        ".....#####..........",
        ".################...",
        ".##################.",
        ".##################.",
        ".################...",
        ".....#####..........",
        "......####..........",
        "....................",
    ],
    "gear": [
        ".........##.........",
        ".........##.........",
        "....#....##....#....",
        ".....#........#.....",
        "......########......",
        "...###oooooooo#####.",
        "..#ooooo######ooooo#",
        "..#oooo#......#oooo#",
        "..#ooo#........#ooo#",
        "..#ooo#........#ooo#",
        "..#ooo#........#ooo#",
        "..#oooo#......#oooo#",
        "..#ooooo######ooooo#",
        "...###oooooooo#####.",
        "......########......",
        ".....#........#.....",
        "....#....##....#....",
        ".........##.........",
        ".........##.........",
        "....................",
    ],
    "network": [
        "....................",
        ".........##.........",
        ".........##.........",
        "........#oo#........",
        ".......#oooo#.......",
        "........#oo#........",
        ".........##.........",
        ".........##.........",
        "...##.....##.....##.",
        "..#oo#....##....#oo#",
        "..#ooo##..##..##ooo#",
        "..#oooo#..##..#oooo#",
        "...#oo#...##...#oo#.",
        "....##....##....##..",
        "....##..........##..",
        "....##..........##..",
        "....##..........##..",
        "...####........####.",
        "....................",
        "....................",
    ],
    "image": [
        "....................",
        ".##################.",
        ".#o##############o#.",
        ".##..............##.",
        ".##...####.......##.",
        ".##..#oooo#......##.",
        ".##...####.......##.",
        ".##..............##.",
        ".##..........#####..",
        ".##........##oooo#..",
        ".##......##oooooo#..",
        ".##....##oooooooo#..",
        ".##..##oooooooooo#..",
        ".#o##ooooooooooooo#.",
        ".##################.",
        "....................",
        "....................",
        "....................",
        "....................",
        "....................",
    ],
    "music": [
        "....................",
        "..........########..",
        "..........#########.",
        "..........##.....##.",
        "..........##......#.",
        "..........##........",
        "..........##........",
        "..........##........",
        "..........##........",
        "..........##........",
        "..........##........",
        ".....######.........",
        "....########........",
        "...##########.......",
        "...##########.......",
        "....########........",
        ".....######.........",
        "....................",
        "....................",
        "....................",
    ],
    "dice": [
        "....................",
        ".##################.",
        "####################",
        "####################",
        "###..............###",
        "###..###.....###.###",
        "###..###.....###.###",
        "###..###.....###.###",
        "###..............###",
        "########..##########",
        "########..##########",
        "###..............###",
        "###..###..###....###",
        "###..###..###....###",
        "###..###..###....###",
        "###..............###",
        "####################",
        "####################",
        ".##################.",
        "....................",
    ],
    "book": [
        "....................",
        ".####............###",
        ".######........#####",
        ".########....#######",
        ".###################",
        ".###################",
        ".###################",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".#####..........####",
        ".###################",
        ".###################",
        "....................",
    ],
    "terminal": [
        "....................",
        ".##################.",
        ".##################.",
        ".##..............##.",
        ".##..##..........##.",
        ".##...##.........##.",
        ".##....##........##.",
        ".##...##.........##.",
        ".##..##..........##.",
        ".##..............##.",
        ".##....#######...##.",
        ".##..............##.",
        ".##..............##.",
        ".##################.",
        ".##################.",
        "....................",
        "....................",
        "....................",
        "....................",
        "....................",
    ],
    "shield": [
        "....................",
        "...##############...",
        "..################..",
        ".##################.",
        ".##################.",
        ".##################.",
        ".####..######..####.",
        "..####..####..####..",
        "...#############....",
        "....############....",
        ".....##########.....",
        "......########......",
        ".......######.......",
        "........####........",
        ".........##.........",
        "....................",
        "....................",
        "....................",
        "....................",
        "....................",
    ],
    "clock": [
        "....................",
        "......########......",
        "....############....",
        "...##############...",
        "..################..",
        "..################..",
        "..######..########..",
        "..######...#######..",
        "..######....######..",
        "..######.....#####..",
        "..################..",
        "..################..",
        "..################..",
        "..################..",
        "...##############...",
        "....############....",
        "......########......",
        "....................",
        "....................",
        "....................",
    ],
    "chart": [
        "....................",
        "....................",
        "....................",
        "...##...............",
        "...##.....##........",
        "...##.....##........",
        "...##.....##..##....",
        "...##.....##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "...##..##.##..##....",
        "..#oo##oo#oo##oo##..",
        "..################..",
        "....................",
        "....................",
        "....................",
    ],
    "star": [
        ".........##.........",
        ".........##.........",
        "........####........",
        "........####........",
        ".......######.......",
        ".##################.",
        "..################..",
        "...##############...",
        "....############....",
        ".....##########.....",
        ".....##########.....",
        "....############....",
        "....###..##..###....",
        "...###....##...###..",
        "...##......##...##..",
        "..##........##...##.",
        "..#..........#....#.",
        "....................",
        "....................",
        "....................",
    ],
}

_GLYPH_KEYWORDS = [
    ("key", ("cryptex", "cipher", "encrypt", "decrypt", "encode", "decode", "key")),
    ("hash", ("hash", "md5", "sha1", "sha256", "sha512", "digest")),
    ("lock", ("password", "vault", "lock", "secret")),
    ("swap", ("translate", "convert", "swap", "exchange")),
    ("music", ("music", "audio", "song", "sound", "play")),
    ("image", ("image", "photo", "picture", "camera")),
    ("dice", ("dice", "random", "game", "fun", "joke")),
    ("book", ("book", "knowledge", "docs", "read", "library")),
    ("network", ("mesh", "network", "net", "peer", "sync")),
    ("shield", ("scan", "security", "shield", "guard", "nmap", "vuln")),
    ("terminal", ("shell", "terminal", "command", "cli", "exec")),
    ("clock", ("clock", "time", "schedule", "timer", "alarm")),
    ("chart", ("chart", "stats", "graph", "metric", "report")),
    ("text", ("text", "word", "string", "note", "write")),
    ("gear", ("tool", "config", "setting", "util")),
    ("star", ("oracle", "fortune", "magic", "star")),
]


def valid_icon_art(raw):
    # Author-supplied face art (Build 3, issue #101): a manifest's
    # "icon" — exactly 20 strings of exactly 20 characters over
    # {'.', '#', 'o'} ('#' accent fill, 'o' shade fill, '.' empty),
    # the same pixel language as GLYPH_PIXELS. Returns the rows
    # normalized (a fresh list of plain strings) or None when the
    # art is absent or malformed in any way. Callers decide what
    # None means: a forge refuses in plain words; a picture parse
    # treats bad art as no art, so a damaged icon never bricks a
    # cartridge — it falls back to the glyph.
    if not isinstance(raw, list) or len(raw) != 20:
        return None
    rows = []
    for row in raw:
        if not isinstance(row, str) or len(row) != 20:
            return None
        if any(c not in ".#o" for c in row):
            return None
        rows.append(row)
    return rows


ICON_ART_REFUSAL = (
    "The icon art is not in the shape a cartridge face draws: "
    "20 rows of 20 characters each, using only . (empty), "
    "# (accent) and o (shade). Fix the icon in the manifest, "
    "or take it out and let the chip wear its glyph."
)


def _icon_art_of(payload):
    # The payload's own declared art, validated — None when the
    # payload carries none or carries a damaged one.
    if not isinstance(payload, dict):
        return None
    return valid_icon_art(payload.get("icon"))


def _manifest_icon(manifest):
    # A repo manifest's declared icon, for the install fetch: a
    # manifest that declares art in the wrong shape refuses the
    # install in plain words (the same refusal a forge gives) —
    # silently dropping the author's art would forge a face they
    # never approved. No icon declared is simply no icon.
    raw = (manifest or {}).get("icon")
    if not raw:
        return []
    rows = valid_icon_art(raw)
    if rows is None:
        raise ValueError(ICON_ART_REFUSAL)
    return rows


def _payload_icon(data):
    # An endpoint body's declared icon: (rows, error) — rows is []
    # when no art was sent; error is the plain-words refusal when
    # art was sent in the wrong shape.
    raw = (data or {}).get("icon")
    if not raw:
        return [], ""
    rows = valid_icon_art(raw)
    if rows is None:
        return [], ICON_ART_REFUSAL
    return rows, ""


def _tile_field_seed(name):
    # The legacy pattern's seed discipline: 7 folded with the
    # name's characters — the window's field and the seeded
    # fallback share one seed so a cartridge's texture is stable
    # across forges of the same name.
    seed = 7
    for ch in str(name or ""):
        seed = (seed * 31 + ord(ch)) & 0xFFFFFFFF
    return seed


def _draw_tile_field(rect, wx, wy, win, name, rgb, window_rgb, skip=None):
    # The window's background texture (Build 3, issue #101): a
    # QR-style tile field on the window's own 20x20 grid — tiles
    # seeded from the cartridge's name with the legacy pattern's
    # LCG, and QR finder squares in three corners (a 7x7 finder:
    # filled border, one-cell gap, filled 3x3 core). Drawn in the
    # face accent dimmed to a texture, behind the glyph. Cells the
    # silhouette will stamp (skip) take the field UNDER the art
    # instead of the finder squares: the glyph always covers the
    # finders, so letting them draw there only punched two big
    # holes through every solid mass the field was meant to sit
    # behind.
    cell = win // 20
    seed = _tile_field_seed(name)
    skip = skip or set()

    def finder(fx, fy):
        rect(wx + fx * cell, wy + fy * cell,
             wx + (fx + 7) * cell, wy + (fy + 7) * cell, rgb)
        rect(wx + (fx + 1) * cell, wy + (fy + 1) * cell,
             wx + (fx + 6) * cell, wy + (fy + 6) * cell, window_rgb)
        rect(wx + (fx + 2) * cell, wy + (fy + 2) * cell,
             wx + (fx + 5) * cell, wy + (fy + 5) * cell, rgb)

    def in_finder(cx, cy):
        return ((cx < 8 and cy < 8) or (cx > 11 and cy < 8)
                or (cx < 8 and cy > 11))

    for cy in range(20):
        for cx in range(20):
            seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF
            if (cx, cy) in skip or in_finder(cx, cy):
                continue
            if seed / 0x7FFFFFFF > 0.52:
                rect(wx + cx * cell, wy + cy * cell,
                     wx + (cx + 1) * cell, wy + (cy + 1) * cell, rgb)
    finder(0, 0)
    finder(13, 0)
    finder(0, 13)


def _draw_spray(rect, wx, wy, win, name, accent, shade, field, dark, deep):
    # The knowledge cartridge's window face (issue #117): the
    # seeded tile spray grown to fill the whole window — no
    # glyph. Same seed discipline as the tile field (the
    # cartridge's name through the legacy LCG), on the window's
    # 20x20 grid. Cells land in the family accent, its shade
    # (x0.45), its field tone (x0.30), or dark; the fill runs
    # denser and brighter toward the middle and thins toward
    # the rim, the way the field-forged sample face reads.
    cell = win // 20
    seed = _tile_field_seed(name)

    def rnd():
        nonlocal seed
        seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF
        return seed / 0x7FFFFFFF

    for cy in range(20):
        for cx in range(20):
            dx = (cx - 9.5) / 9.5
            dy = (cy - 9.5) / 9.5
            d = min(1.0, (dx * dx + dy * dy) ** 0.5)
            if rnd() > 0.88 - 0.5 * d:
                rgb = deep if rnd() < 0.22 else dark
            else:
                t = rnd()
                if t < 0.55 - 0.30 * d:
                    rgb = accent
                elif t < 0.82:
                    rgb = shade
                else:
                    rgb = field
            rect(wx + cx * cell, wy + cy * cell,
                 wx + (cx + 1) * cell, wy + (cy + 1) * cell, rgb)


def _draw_silhouette(rect, wx, wy, win, grid, accent, shade):
    # A 20x20 silhouette over the window's field, drawn like the
    # Mesh renderer's own stamp: the art's cells stamp at full
    # window size and the field shows only OUTSIDE the art — the
    # stamp overwrites whole cells, finders included, and the
    # field is never allowed to punch holes through a solid mass.
    # '#' cells in the full accent, 'o' cells in the shade tone,
    # and a shade offset copy beneath for depth. (An earlier
    # version redrew the field INSIDE the art and the two read as
    # one texture — the object dissolved; the stamp is what keeps
    # it an object.)
    cell = win // 20
    off = (win - cell * 20) // 2
    lift = max(3, cell // 3)
    for passes in ((1, shade), (0, None)):
        for cy in range(20):
            for cx in range(20):
                ch = grid[cy][cx]
                if ch == ".":
                    continue
                if passes[0]:
                    rgb = shade
                    dx = dy = lift
                else:
                    rgb = accent if ch == "#" else shade
                    dx = dy = 0
                rect(wx + off + cx * cell + dx, wy + off + cy * cell + dy,
                     wx + off + (cx + 1) * cell + dx,
                     wy + off + (cy + 1) * cell + dy, rgb)


def pick_chip_glyph(payload):
    # The face a payload earns: its own declared glyph first, then
    # the keyword heuristic over name + description, else None —
    # and None is what keeps the legacy seeded face.
    g = str((payload or {}).get("glyph") or "").strip().lower()
    if g in GLYPH_PIXELS:
        return g
    hay = (
        str((payload or {}).get("name") or "")
        + " "
        + str((payload or {}).get("description") or "")
    ).lower()
    for name, words in _GLYPH_KEYWORDS:
        if any(w in hay for w in words):
            return name
    return None


def _cartridge_face_png(payload, keyword):
    # The card-grade family face (0.1.9e Build 2; window reworked
    # in Build 3, issue #101): the cartridge drawn with the Mesh
    # pairing cards' machinery — the same block font, pixel buffer,
    # and PNG assembly — at card resolution in the classic portrait
    # stance, 528x616 (the old 96x112 face at 5.5x, same proportions).
    # Family DNA intact: black shell, rim, family-accent notched bands top
    # and bottom as the cartridge's contacts (notches on the bands'
    # inner edges), the thing's art large in its window, then its
    # name, its use line, and honest small print. Build 3's window:
    # a dim QR-style tile field textures the window, and the glyph
    # — or the author's own declared "icon" art, which outranks it —
    # renders over it as a solid silhouette with a shade offset. A
    # chip no art claims keeps the legacy name-seeded pattern,
    # drawn plain in the window as before. Faces are minted at
    # forge time — installed and published pictures are never
    # rewritten.
    W, H = 528, 616
    BG = (10, 14, 18)
    SHELL = (8, 9, 11)
    RIM = (34, 48, 61)
    CORAL = (255, 111, 97)
    # Color is family (issue #117): chips keep the coral red;
    # knowledge packs wear the presence green (52, 211, 153 —
    # the app's own #34d399); the Mesh pairing cards wear
    # their blue in _mesh_card_png_render. Every derivation
    # below — bands, header, shade (x0.45), field (x0.30),
    # silhouette, spray — follows the family's accent.
    FACE_ACCENTS = {"navi-chip": CORAL, "navi-rag": (52, 211, 153)}
    ACCENT = FACE_ACCENTS.get(keyword, CORAL)
    WINDOW = (14, 17, 21)
    INK = (230, 241, 245)
    DIM = (159, 179, 200)
    img = bytearray(bytes(SHELL) * (W * H))

    def px(x, y, rgb):
        if 0 <= x < W and 0 <= y < H:
            o = (y * W + x) * 3
            img[o] = rgb[0]
            img[o + 1] = rgb[1]
            img[o + 2] = rgb[2]

    def rect(x0, y0, x1, y1, rgb):
        for yy in range(y0, y1):
            for xx in range(x0, x1):
                px(xx, yy, rgb)

    for i in range(4):
        for x in range(W):
            px(x, i, RIM)
            px(x, H - 1 - i, RIM)
        for y in range(H):
            px(i, y, RIM)
            px(W - 1 - i, y, RIM)

    BAND_H = 26
    TOP_BAND_Y = 24
    BOT_BAND_Y = H - 24 - BAND_H

    def band(y0, notch_first):
        # A family-accent contact band; along its inner edge a stepped
        # bite is taken out of every second cell — the notches
        # the small face drew, at card scale.
        rect(4, y0, W - 4, y0 + BAND_H, ACCENT)
        for x in range(4, W - 4):
            if (x // 44) % 2 == 0:
                for yy in notch_first:
                    px(x, yy, SHELL)

    band(TOP_BAND_Y, (TOP_BAND_Y + BAND_H - 2, TOP_BAND_Y + BAND_H - 1))
    band(BOT_BAND_Y, (BOT_BAND_Y, BOT_BAND_Y + 1))

    MX = 40
    if keyword == "navi-rag":
        head_l, head_r = "KNOWLEDGE", "NAVI-RAG"
    else:
        head_l, head_r = "APEX CHIP", "NAVI-CHIPS"
    y = TOP_BAND_Y + BAND_H + 22
    _mesh_face_draw(img, W, H, MX, y, head_l, 3, ACCENT)
    _mesh_face_draw(
        img, W, H, W - MX - _mesh_face_text_w(head_r, 3), y, head_r, 3, DIM
    )
    y += 21 + 22

    # The glyph window: rim-framed, dark inside. Build 3's
    # treatment: a dim QR-style tile field as the window's texture,
    # and the art as a solid silhouette over it — the author's own
    # declared icon first, then the built-in glyph (picked by name
    # or keyword, chips only). A knowledge cartridge's window is
    # the seeded spray instead (issue #117) unless the author's
    # icon claims it. The shade tone is the family accent
    # darkened; author art's 'o' cells draw in it. A chip no art
    # claims keeps the legacy seeded pattern, plain.
    WIN = 300
    wx = (W - WIN) // 2
    wy = y
    rect(wx - 6, wy - 6, wx + WIN + 6, wy + WIN + 6, RIM)
    rect(wx, wy, wx + WIN, wy + WIN, WINDOW)
    art = _icon_art_of(payload)
    glyph = None
    if art is None and keyword != "navi-rag":
        glyph = pick_chip_glyph(payload)
        if glyph is not None and glyph in GLYPH_PIXELS:
            art = GLYPH_PIXELS[glyph]
    if keyword == "navi-rag" and art is None:
        # A knowledge cartridge's center is the spray (issue
        # #117): the seeded tile field grown to fill the whole
        # window, in the green family — it outranks even a
        # declared glyph. Only the author's own icon art
        # (handled by the silhouette branch below) outranks it.
        shade = tuple(int(c * 0.45) for c in ACCENT)
        field = tuple(int(c * 0.30) for c in ACCENT)
        _draw_spray(rect, wx, wy, WIN, payload.get("name"),
                    ACCENT, shade, field, WINDOW, BG)
    elif art is not None:
        shade = tuple(int(c * 0.45) for c in ACCENT)
        field = tuple(int(c * 0.30) for c in ACCENT)
        # The art's own halo: the silhouette dilated by one cell in
        # the window color, drawn FIRST — it clears the field out
        # of the object's neighborhood so the eye meets a dark halo,
        # then the silhouette, never field-against-accent noise.
        # Behind it, the whole window sits on the shade's backplate
        # so the texture reads as one woven field, the way the
        # Mesh renderer's cards carry their accent field.
        _cell = WIN // 20
        _filled = {(cx, cy) for cy in range(20) for cx in range(20)
                   if art[cy][cx] != "."}
        _halo = set()
        for (cx, cy) in _filled:
            for dx in (-1, 0, 1):
                for dy in (-1, 0, 1):
                    if 0 <= cx + dx < 20 and 0 <= cy + dy < 20:
                        _halo.add((cx + dx, cy + dy))
        rect(wx, wy, wx + WIN, wy + WIN, shade)
        for (cx, cy) in sorted(_halo):
            rect(wx + cx * _cell, wy + cy * _cell,
                 wx + (cx + 1) * _cell, wy + (cy + 1) * _cell, WINDOW)
        _draw_tile_field(rect, wx, wy, WIN, payload.get("name"), field, WINDOW,
                         skip=_filled)
        _draw_silhouette(rect, wx, wy, WIN, art, ACCENT, shade)
    else:
        if keyword == "navi-rag":
            palette = [(52, 211, 153), (45, 180, 130), (74, 222, 180)]
        else:
            palette = [
                (34, 211, 238), (245, 181, 68), (52, 211, 153),
                (248, 113, 113), (167, 139, 250),
            ]
        h = 0
        seed = 7
        for ch in str(payload.get("name") or ""):
            h = (h * 31 + ord(ch)) & 0xFFFFFFFF
            seed = (seed * 31 + ord(ch)) & 0xFFFFFFFF
        accent = palette[h % len(palette)]
        blocks = []
        for _ in range(64):
            seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF
            blocks.append(seed / 0x7FFFFFFF > 0.55)
        pad = 24
        cell = (WIN - 2 * pad) / 8.0
        for by in range(8):
            for bx in range(8):
                if blocks[by * 8 + bx]:
                    rect(
                        int(wx + pad + bx * cell), int(wy + pad + by * cell),
                        int(wx + pad + (bx + 1) * cell),
                        int(wy + pad + (by + 1) * cell),
                        accent,
                    )
                else:
                    rect(
                        int(wx + pad + bx * cell), int(wy + pad + by * cell),
                        int(wx + pad + (bx + 1) * cell),
                        int(wy + pad + (by + 1) * cell),
                        BG,
                    )
    y = wy + WIN + 24

    # The name band: auto-fitting like the Mesh cards' name —
    # start big, shrink until it fits, truncate only at the floor.
    name = _mesh_face_clean(payload.get("name") or "") or (
        "KNOWLEDGE PACK" if keyword == "navi-rag" else "CHIP"
    )
    scale = 7
    while scale > 2 and _mesh_face_text_w(name, scale) > W - 2 * MX:
        scale -= 1
    if _mesh_face_text_w(name, scale) > W - 2 * MX:
        name = name[: max(0, (W - 2 * MX) // (6 * scale) - 3)] + "..."
    _mesh_face_draw(img, W, H, MX, y, name, scale, INK)
    y += 7 * scale + 16

    # The use line: the description, wrapped at the width that
    # fits scale 2, up to three lines.
    max_chars = (W - 2 * MX) // (6 * 2)
    use = _mesh_face_wrap(
        _mesh_face_clean(payload.get("description") or ""), max_chars
    )
    for line in use[:3]:
        if y + 14 > BOT_BAND_Y - 44:
            break
        _mesh_face_draw(img, W, H, MX, y, line, 2, DIM)
        y += 14 + 8

    # Footer small print, above the bottom band with clearance:
    # the version when the picture's own data knows it (never
    # invented), the output kind, and the family mark. A
    # knowledge cartridge's footer carries its version and its
    # document count instead.
    foot_bits = []
    ver = str(payload.get("version") or "").strip()
    if keyword == "navi-rag":
        docs = payload.get("documents")
        if isinstance(docs, list):
            foot_bits.append(
                str(len(docs)) + (" DOCUMENT" if len(docs) == 1 else " DOCUMENTS")
            )
        foot_bits.append("NAVI-RAG")
    else:
        # The output KIND, not the output text: a chip whose
        # run() produces text wears TEXT; one that only declares
        # an output wears the first words of it (the way a
        # Mesh card reads its offer).
        if str(payload.get("code") or "").strip():
            foot_bits.append("TEXT")
        else:
            out_line = _mesh_face_clean(payload.get("output") or "")
            if out_line:
                foot_bits.append(out_line[:18])
        foot_bits.append("NAVI-CHIPS")
    foot = _mesh_face_clean(" - ".join(foot_bits))
    # The version rides bottom-RIGHT (issue #117, layout lock):
    # its own element on the same baseline as the left footer,
    # omitted when the picture's data does not know it (never
    # invented). The left footer truncates toward the version,
    # never into it — the version's width is reserved first.
    ver_txt = _mesh_face_clean("V" + ver.upper()) if ver else ""
    ver_w = _mesh_face_text_w(ver_txt, 2) if ver_txt else 0
    left_max = W - 2 * MX - (ver_w + 24 if ver_txt else 0)
    while foot and _mesh_face_text_w(foot, 2) > left_max:
        foot = foot[:-1]
    _mesh_face_draw(img, W, H, MX, BOT_BAND_Y - 14 - 16, foot, 2, DIM)
    if ver_txt:
        _mesh_face_draw(img, W, H, W - MX - ver_w, BOT_BAND_Y - 14 - 16, ver_txt, 2, DIM)

    raw = bytearray()
    for yy in range(H):
        raw.append(0)
        raw += img[yy * W * 3:(yy + 1) * W * 3]
    ihdr = struct.pack(">IIBBBBB", W, H, 8, 2, 0, 0, 0)
    return (
        b"\x89PNG\r\n\x1a\n"
        + _png_chunk(b"IHDR", ihdr)
        + _png_chunk(b"IDAT", zlib.compress(bytes(raw)))
        + _png_chunk(b"IEND", b"")
    )


def build_cartridge_png(payload):
    # The PICO-8 trick in Battle Network dress: the visible image is
    # a battle-chip label; the chip's whole codebase rides inside the
    # PNG in a tEXt chunk (keyword "navi-chip", JSON payload).
    face = _cartridge_face_png(payload, "navi-chip")
    text = b"navi-chip\x00" + json.dumps(payload).encode()
    # Rebuild with the payload chunk ahead of the image data.
    ihdr_end = 8 + 12 + 13
    return (
        face[:ihdr_end]
        + _png_chunk(b"tEXt", text)
        + face[ihdr_end:]
    )


def _valid_chip_files(raw, code):
    # A chip's file set as [{path, content}]: root.py first, holding
    # the entry code; then each safe module file (relative .py path,
    # no "..", no leading slash). Unsafe paths and duplicates are
    # dropped. A chip with no file set reads as a lone root.py.
    files = [{"path": "root.py", "content": code}]
    seen = {"root.py"}
    if isinstance(raw, list):
        for item in raw:
            if not isinstance(item, dict):
                continue
            path = str(item.get("path") or "")
            parts = path.split("/")
            if (not path or path.startswith("/") or ".." in parts
                    or not path.endswith(".py") or path in seen):
                continue
            seen.add(path)
            files.append({"path": path, "content": str(item.get("content") or "")})
    return files


def parse_cartridge_png(data):
    # Reads the "navi-chip" tEXt payload back out of a chip PNG —
    # the inverse of build_cartridge_png.
    if len(data) < 8 or data[:8] != b"\x89PNG\r\n\x1a\n":
        return None
    pos = 8
    while pos + 12 <= len(data):
        (length,) = struct.unpack(">I", data[pos:pos + 4])
        kind = data[pos + 4:pos + 8]
        chunk_data = data[pos + 8:pos + 8 + length]
        if len(chunk_data) < length:
            return None
        if kind == b"tEXt":
            nul = chunk_data.find(b"\x00")
            if nul > 0 and chunk_data[:nul] == b"navi-chip":
                try:
                    payload = json.loads(chunk_data[nul + 1:].decode("utf-8", "replace"))
                except Exception:
                    return None
                if isinstance(payload, dict) and payload.get("name"):
                    code = str(payload.get("code") or "")
                    return {
                        "name": str(payload.get("name") or ""),
                        "description": str(payload.get("description") or ""),
                        "code": code,
                        "output": str(payload.get("output") or ""),
                        "files": _valid_chip_files(payload.get("files"), code),
                        "glyph": str(payload.get("glyph") or ""),
                        # Build 3 (#101): the author's declared
                        # icon art rides the same way. A damaged
                        # icon parses as NO icon — the face falls
                        # back to the glyph, and a bad picture of
                        # art never bricks the cartridge.
                        "icon": _icon_art_of(payload) or [],
                        # Issue #99: the version the picture itself
                        # carries (Build 2 and later forges); older
                        # pictures simply have none — "" here, and
                        # no surface invents one for them.
                        "version": str(payload.get("version") or ""),
                    }
                return None
        if kind == b"IEND":
            break
        pos += 8 + length + 4
    return None


def deck_contents(cfg):
    # Everything that rides a picture, for ChipDeck (0.1.9e, #94):
    # installed chips, the Mesh pairing cards on disk, and the
    # knowledge packs (each pack's cartridge is built on demand by
    # /api/rag/cartridge.png). State is read from the same stores
    # the owning pages read — the Deck is a view, not a store.
    chips = [
        {
            "file": c["file"], "name": c["name"],
            "description": c.get("description") or "",
            "glyph": pick_chip_glyph(c) or "",
            # Issue #99: the version the picture itself carries —
            # "" for pictures forged before it was carried, and
            # the Deck shows nothing rather than a guess for them.
            "version": str(c.get("version") or ""),
            "source": c.get("source"), "sourceKey": c.get("sourceKey"),
            "locallyEdited": bool(c.get("locallyEdited")),
            "unreadable": bool(c.get("unreadable")),
        }
        for c in local_chips()
    ]
    cards = []
    try:
        ms = mesh_settings(cfg)
        paired_ids = {
            str(p.get("nodeId") or "")
            for p in (ms["borrow"]["peers"] or [])
            if isinstance(p, dict)
        }
    except Exception:
        paired_ids = set()
    try:
        card_files = sorted(os.listdir(MESH_CARDS_DIR))
    except Exception:
        card_files = []
    for fn in card_files:
        if not fn.lower().endswith(".png"):
            continue
        entry = {"file": fn, "name": fn[:-4], "nodeId": "", "own": fn == "own.png", "paired": False}
        try:
            with open(os.path.join(MESH_CARDS_DIR, fn), "rb") as f:
                peer, _e = parse_mesh_card_png(f.read())
            if peer is not None:
                entry["name"] = peer.get("name") or entry["name"]
                entry["nodeId"] = str(peer.get("nodeId") or "")
                entry["paired"] = entry["nodeId"] in paired_ids
        except Exception:
            pass
        cards.append(entry)
    packs = [
        {
            "slug": p["slug"], "name": p["name"],
            "description": p.get("description") or "",
            "fullName": p["fullName"], "active": bool(p["active"]),
            "docCount": p["docCount"],
        }
        for p in local_rag_packs()
    ]
    return {"chips": chips, "meshCards": cards, "ragPacks": packs}


def _png_text_keywords(data):
    keys = []
    if not isinstance(data, (bytes, bytearray)) or len(data) < 8 or bytes(data[:8]) != b"\x89PNG\r\n\x1a\n":
        return keys
    pos = 8
    while pos + 12 <= len(data):
        (length,) = struct.unpack(">I", data[pos:pos + 4])
        kind = data[pos + 4:pos + 8]
        if pos + 12 + length > len(data):
            break
        if kind == b"tEXt":
            chunk_data = data[pos + 8:pos + 8 + length]
            nul = chunk_data.find(b"\x00")
            if nul > 0:
                keys.append(chunk_data[:nul].decode("utf-8", "replace"))
        if kind == b"IEND":
            break
        pos += 8 + length + 4
    return keys


def local_chips():
    # Every chip in chips/ is a chip loaded on this machine.
    # The install registry supplies each chip's origin: a
    # GitHub full name, a local/ record for Studio-forged chips, or
    # no record at all for a chip dropped in by hand — and
    # whether a Studio save has edited an installed chip locally.
    by_file = {}
    for r in load_chip_installs():
        if r.get("local_file"):
            by_file[r["local_file"]] = r
    chips = []
    try:
        files = sorted(os.listdir(CHIPS_DIR))
    except Exception:
        files = []
    for fn in files:
        if not fn.lower().endswith(".png"):
            continue
        rec = by_file.get(fn)
        full = (rec or {}).get("full_name") or ""
        try:
            with open(os.path.join(CHIPS_DIR, fn), "rb") as f:
                payload = parse_cartridge_png(f.read())
        except Exception:
            payload = None
        if payload:
            payload["file"] = fn
            payload["slug"] = fn[:-4]
            payload["source"] = "github" if full and not full.startswith("local/") else "local"
            payload["sourceKey"] = full if payload["source"] == "github" else None
            payload["locallyEdited"] = bool((rec or {}).get("locally_edited"))
            if not payload.get("requirements"):
                payload["requirements"] = list((rec or {}).get("requirements") or [])
            chips.append(payload)
        else:
            # Build .86 (issue #79): a cartridge that fails to
            # parse is never silently dropped — it surfaces as an
            # unreadable entry (filename for a name, empty code)
            # so the Local tab can show it honestly, still
            # deletable, instead of the chip simply vanishing.
            chips.append({
                "name": fn[:-4],
                "description": "",
                "code": "",
                "output": "",
                "files": [],
                "file": fn,
                "slug": fn[:-4],
                "source": "github" if full and not full.startswith("local/") else "local",
                "sourceKey": full if full and not full.startswith("local/") else None,
                "locallyEdited": bool((rec or {}).get("locally_edited")),
                "unreadable": True,
                "requirements": list((rec or {}).get("requirements") or []),
            })
    return chips


def match_local_chip(query, chips):
    # Matches a chat query against chip file slugs and names; anything
    # after the match is the chip's argument string.
    q = (query or "").strip().lower()
    if not q:
        return None, ""
    for chip in chips:
        keys = {
            chip["slug"].lower(),
            chip["name"].lower(),
            re.sub(r"[^a-z0-9]+", "-", chip["name"].lower()).strip("-"),
        }
        for key in keys:
            if q == key:
                return chip, ""
            if q.startswith(key + " "):
                return chip, (query or "").strip()[len(key):].strip()
    return None, ""


def _chip_keys(chip):
    return {
        chip["slug"].lower(),
        chip["name"].lower(),
        re.sub(r"[^a-z0-9]+", "-", chip["name"].lower()).strip("-"),
    }


def resolve_chip_invocation(rest, chips):
    # The ./ resolver (0.1.9e, issue #97): "./name args..." with the
    # name resolved in widening rings — the exact slug or name, then
    # the name with a leading "navi-" forgiven on either side
    # ("./cryptex" finds navi-cryptex), then a unique prefix. Returns
    # (chip, args) on a run, (None, candidates) when more than one
    # chip fits, and (None, []) when nothing does.
    text = (rest or "").strip()
    if not text:
        return None, []
    parts = text.split(None, 1)
    token = parts[0].lower()
    args = parts[1] if len(parts) > 1 else ""
    for chip in chips:
        if token in _chip_keys(chip):
            return chip, args
    bare = token[5:] if token.startswith("navi-") else token
    for chip in chips:
        keys = {k[5:] if k.startswith("navi-") else k for k in _chip_keys(chip)}
        if bare in keys:
            return chip, args
    hits = []
    for chip in chips:
        keys = set(_chip_keys(chip)) | {
            k[5:] if k.startswith("navi-") else k for k in _chip_keys(chip)
        }
        if any(k.startswith(token) or k.startswith(bare) for k in keys if k):
            hits.append(chip)
    if len(hits) == 1:
        return hits[0], args
    return None, hits


def chip_shelf_reply(inv):
    # The shelf as a chat reply: every chip on this machine with its
    # one-line description, for a bare "/chip" or a bare "./".
    shelf = local_chips()
    if not shelf:
        return (
            "No chips are installed yet — the shelf is empty. Open the Chips tab and "
            "install one from the GitHub catalog (Navi User Repo) or forge your own in Chip "
            "Studio; then it lands here and " + inv + " <name> runs it."
        )
    lines = []
    for c in shelf:
        d = (c.get("description") or "").strip()
        if len(d) > 90:
            d = d[:87] + "..."
        lines.append("• " + c["name"] + (" — " + d if d else ""))
    return (
        "On the shelf — " + inv + " <name> runs one:\n"
        + "\n".join(lines)
    )


_CHIP_SLASH = re.compile(r"^/chip\s+(.+)$", re.I)
_CHIP_RUN = re.compile(r"^run\s+(?:the\s+)?(.+?)(?:\s+chip)?$", re.I)


def chip_invocation(text):
    # The chip query a chat message carries, if it carries one:
    # "/chip <name>", "run the <name> chip", or the name on its own.
    t = (text or "").strip()
    m = _CHIP_SLASH.match(t)
    if m:
        return m.group(1)
    m = _CHIP_RUN.match(t)
    if m:
        return m.group(1)
    return t


# The navi_internal a chip sees when the bridge runs it: the methods
# the real NaviApp offers chip code, backed by the bridge. print()
# output and print_message both land in the chip's captured output;
# llm_chat proxies to the llamafile with the same (text, status)
# return shape the real chips unpack.
NAVI_INTERNAL_STUB = '''
import os, json, urllib.request

class _NaviInstance:
    def print_message(self, text, include_ai_name=True):
        print(text)
    def llm_chat(self, user_message, called_from_app=False):
        endpoint = os.environ.get("NAVI_LLAMAFILE_ENDPOINT", "http://localhost:8080/v1")
        model = os.environ.get("NAVI_MODEL", "navi")
        if not str(user_message or "").strip():
            # (#224) An empty ask is refused here, by name — the
            # model server rejects empty content outright, and a
            # chip deserves the reason, not the raw refusal.
            return ("(llm_chat was given nothing to ask — the message was empty.)", 0)
        body = json.dumps({"model": model, "messages": [{"role": "user", "content": user_message}], "stream": False, "max_tokens": 512}).encode()
        req = urllib.request.Request(endpoint + "/chat/completions", data=body, headers={"Content-Type": "application/json"})
        try:
            with urllib.request.urlopen(req, timeout=30) as resp:
                data = json.loads(resp.read().decode())
            return data["choices"][0]["message"]["content"], 200
        except Exception as e:
            return ("(llm_chat could not reach the model server: " + str(e) + ")", 0)
    def process_message(self, user_message):
        print("(process_message is not available during a bridge chip run)")
    def get_user(self):
        return os.environ.get("USER") or os.environ.get("USERNAME") or "operator"
    def set_user(self, sys_user):
        pass
    def clear_terminal(self):
        pass
    def chat_with_navi(self):
        print("(chat_with_navi is not available during a bridge chip run)")
    def setup_history(self):
        pass
    def setup_navi_vocab(self):
        pass

navi_instance = _NaviInstance()
NaviApp = _NaviInstance
'''

# The navi.py compatibility shim (build .86, issue #79): chips
# written for the original Navi import navi and call
# get_parameters(text). The shim rides the same stub navi_internal
# uses and restores the old semantics exactly: get_parameters
# returns the shlex-split token list with the command name
# INCLUDED — the chip's run() pops it, the way the old chips were
# written. Written beside every sandbox unless the chip ships its
# own navi.py.
NAVI_PY_SHIM = '''
import shlex
from navi_internal import navi_instance, NaviApp

def get_parameters(text):
    try:
        return shlex.split(text or "")
    except Exception:
        return (text or "").split()
'''

