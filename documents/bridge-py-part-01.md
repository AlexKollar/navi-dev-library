# bridge.py — part 1 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 1–4331 of bridge.py.
Top-level symbols in this part: shell_cwd, load_config, save_config, public_settings, set_public_restart, schedule_bridge_restart, generate_access_code, ensure_access_code, public_token, door_devices, _door_save_devices, _door_prune, door_approved_devices, door_device_for_token, door_cookie_value, door_gate_acceptance, door_touch_device, _door_new_device, door_create_approved, door_register_pending, door_gate_outcome, door_pending_poll, door_approve, door_deny, door_revoke, door_cookie_header, door_devices_payload, lan_ipv4, _ip_is_private_or_cgnat, wan_ipv4, public_state_payload, _gate_escape, _gate_stars_svg, _gate_ring_svg, gate_page_html, gate_pending_html, dhd_settings, dhd_save, dhd_cert_meta, dhd_write_meta, dhd_cert_expiry, dhd_generate_self_signed, dhd_ensure_server_key, _b64u, _acme_http, _der_read_len, _der_read_tlv, _rsa_jwk_from_private_key, _acme_jws, acme_obtain, dhd_gate_page, dhd_throttle_state, dhd_throttle_fail, dhd_throttle_clear, DhdProxyHandler, dhd_start_proxy, dhd_stop_proxy, dhd_self_check, dhd_domain_resolves_here, dhd_bring_up, _dhd_on_summary, dhd_state_payload, dhd_renewal_loop, _dhd_env_set, detect_distro_family, _python3_path, dhd_lowport_commands, dhd_manual_commands, dhd_failure_commands, _dhd_port_state, dhd_prereq_check, _dhd_install_argv, _dhd_run_install_command, dhd_install_prereqs, _ops_load, _ops_save, op_snapshot, op_latest, op_new, op_set.


#!/usr/bin/env python3
"""Navi bridge — serves the Apex dashboard and gives it real
local powers.

Stdlib only. Endpoints:
  GET  /              the dashboard (app/index.html)
  GET  /api/status    backend, model, version, llamafile reachability
  GET  /api/update/check -> the Updater's real check: the newest
                      GitHub release on the install's channel
                      (stable = latest full release; edge = newest
                      incl. pre-releases; ?channel= overrides the
                      saved choice) compared against the VERSION
                      file, with the running version's own release
                      page riding along for the panel's dropdown;
                      a check that cannot complete answers
                      ok: False with the reason
  POST /api/settings/updater-channel {channel} -> persists the
                      Updater's Stable / Edge channel choice in
                      the config's updater block
  POST /api/update/apply -> starts the end-to-end apply: a worker
                      downloads the latest release's package zip
                      (unwrapping a gzip-wrapped payload first),
                      verifies it is an Apex package, stages it
                      under updates/<tag>/package, then hands the
                      install to a detached helper (written into
                      updates/<tag>/) that outlives this bridge,
                      runs the staged installer (never --fresh),
                      relaunches the app through start.sh, and
                      records the outcome in config/update-result.json
  GET  /api/update/result -> the apply's live record: the step it
                      is on while it runs, and afterwards either
                      the confirmed version before → after or the
                      failed step, its reason, and the manual
                      fallback (staged folder + install command)
  GET  /api/commands  the commands actually installed on this machine
                      (a scan of PATH) — feeds command autocomplete
  GET  /api/persona   Navi's Soul and Memory documents (config files)
  POST /api/persona   saves edits to those documents
  POST /api/chat      {messages:[...], sudo, operator_name} -> proxies
                      to the llamafile server, with Navi's persona as
                      the system prompt
  POST /api/shell     {command:"...", sudo} -> runs it, returns
                      output. Commands run from the operator's home
                      directory; elevated commands ride sudo mode
                      (below).
  POST /api/sudo/arm  {password} -> warms sudo's own timestamp once
                      (sudo -v) and enables sudo mode; the password
                      is never stored, logged or synced
  POST /api/sudo/disarm -> disables sudo mode and kills the
                      timestamp (sudo -k), instantly and totally
  GET  /api/sudo/audit -> the append-only record of the elevated
                      commands Navi has run (never rides Sync)
  POST /api/shell/session/open|input|close, GET
       /api/shell/session/state -> the per-chat interactive shell
                      sessions (CLI Mode, below)
  GET  /api/github    -> GitHub account state: repo/issues URLs,
                      whether an OAuth client id is configured,
                      sign-in state, backup settings, last
                      backup/import records (Settings → GitHub &
                      Backup reads it)
  GET  /api/github/pulse -> the project pulse for AlexKollar/
                      Apex (description, stars/forks/watchers,
                      latest release, recent commits, open-issue
                      count, last push), live, no auth
  GET  /api/github/issues -> the repo's current open issues, live
                      from the public API, no auth
  POST /api/github/client-id {client_id} -> stores the OAuth App
                      client id (a config value, not a secret)
  POST /api/github/settings {backup_repo, backup_private,
                      backup_items, backup_interval_days}
  POST /api/github/signin/start -> Device Flow: asks GitHub for a
                      user code the operator enters in their browser
  POST /api/github/signin/poll -> one poll of the token endpoint;
                      on success the token is stored locally only
  POST /api/github/signout -> deletes the local token file
  POST /api/github/backup {dashboard} -> writes the ready-to-fly
                      essentials to the operator's backup repo
  POST /api/github/import/preview {dashboard} -> what an import
                      would change, per essential; writes nothing
  POST /api/github/import/apply -> restores the backup's essentials
                      (only ever called after a preview + confirm)
  GET  /api/sync -> Quick Sync state: the Operator sign-in, the
                      last round's verdict and when it ran (Build 3,
                      issue #176)
  POST /api/sync/run {openChatId} -> one full Quick Sync round:
                      pull, converge, push — chats and Case Files
                      both ways, over the backup repo, riding the
                      Operator identity
  GET  /api/chips/catalog -> the Navi User Repo (NUR) list: a live
                      GitHub search for repos tagged navi-chips,
                      each with its latest release, merged with the
                      installs recorded in chips/installed.json
  GET  /api/chips/readme?repo=<owner/name> -> a chip repo's
                      README as raw Markdown (the NUR flip cards)
  GET  /api/chips/file?name=<file> -> a chip .png from chips/
  POST /api/chips/install {full_name} -> downloads the chip's .py
                      source from the repo at its latest release (or
                      default branch), forges it into a chip in
                      chips/, and records the install in
                      chips/installed.json — the chip joins the
                      folder: pill row, /chip, workflows, execution
  POST /api/chips/update {full_name} -> moves an install to the
                      repo's latest release / push when newer and
                      re-downloads its chip to match
  POST /api/chips/delete {full_name} -> removes the install record
                      and the chip's chip file with it
  POST /api/chips/create {name, description, code, output, files}
                      -> Chip Studio: forges a .png chip into
                      chips/ and registers it as a local install;
                      files is the chip's full file set ([{path,
                      content}] — root.py is the entry, the rest are
                      modules it can import) and rides in the
                      chip beside code
  GET  /api/chips/source?name=<chip file> -> a chip's code
                      and full file set for the Studio editor
                      (local chips)
  POST /api/chips/save {file, name, description, code, output,
                      files} -> saves the edited file set back
                      into the chip
  POST /api/chips/run {query} -> executes a chip's chip code
                      (its run() against a stub of navi_internal)
                      and returns the output; /api/chat runs chips
                      the same way when a message invokes one
  GET  /api/rag/catalog -> the RAG tab's Packs list: a live GitHub
                      search for repos tagged navi-rag, merged with
                      the packs recorded in rag-packs/installed.json
  GET  /api/rag/readme?repo=<owner/name> -> a pack repo's
                      README as raw Markdown (the pack flip cards)
  GET  /api/rag/packs -> {packs}: the pack folders on this machine,
                      with documents, chunk counts, and active flags
  GET  /api/rag/pack-source?slug=<folder> -> a pack's name,
                      description, and documents with bodies, for
                      reopening it in RAG Studio
  POST /api/rag/update-pack {slug, name, description, documents} ->
                      RAG Studio saving an edit into the pack's own
                      folder (documents added/removed), the index
                      rebuilt locally; a GitHub-installed pack is
                      marked locally_edited and its catalog update
                      then refuses rather than overwriting the edits
  POST /api/rag/install {full_name} -> downloads the pack's .md/.txt/.pdf
                      documents from the repo at its latest release
                      (or default branch), builds its index in
                      rag-packs/<slug>/, and records the install
  POST /api/rag/update {full_name} -> moves an install to the repo's
                      latest release / push when newer, re-downloading
                      the documents and rebuilding the index (the
                      active flag survives)
  POST /api/rag/delete {full_name} -> removes the install record and
                      the pack's folder (documents + index) with it
  POST /api/rag/toggle {full_name, active} -> only active packs feed
                      retrieval in /api/chat
  POST /api/rag/forge {name, description, documents|source_dir} ->
                      RAG Studio: packages documents (picked files
                      sent as text — PDFs as base64, their text
                      extracted here — or a folder on this machine's
                      disk) into a local pack in rag-packs/, index
                      built, active and ready to use
  POST /api/rag/export {slug, version} -> builds a GitHub-ready
                      repo ZIP of the pack in rag-exports/: a
                      generated README.md (the pack's GitHub page —
                      installs skip a root README, so it is never
                      indexed), a navi-rag.json manifest, the pack's
                      built index.json (reference copy — installs
                      rebuild it locally from the documents), the
                      documents under documents/, and a .gitignore
  GET  /api/rag/export-file?name=<zip> -> downloads an exported
                      pack repo ZIP from rag-exports/
  POST /api/chips/export {file, version} -> builds a GitHub-ready
                      repo ZIP of a local chip in
                      chips-exports/: a generated README.md, a
                      navi-chip.json manifest, the chip's files at
                      the root (root.py entry + importable modules),
                      the chip .png, and a .gitignore
  GET  /api/chips/export-file?name=<zip> -> downloads an exported
                      chip repo ZIP from chips-exports/
  POST /api/downloads/save {filename, dataBase64} -> writes a
                      download the dashboard fetched inside the
                      desktop shell (chip .png, export ZIP) into
                      ~/Downloads under a sanitized basename —
                      the shell's webview cannot save files itself
  GET  /api/workflows -> {chips, workflows}: the local chips
                      and the saved chains (chips/workflows.json)
  POST /api/workflows/create|delete|run -> manage and run chains;
                      a run feeds each chip's output into the next
  GET  /api/voice     her voice: settings, engine availability, and
                      the live call state (see Voice, below)
  POST /api/voice/settings {enabled, stt_model, tts_engine,
                      tts_voice, rate, pitch} -> saves the voice
                      section of navi.config.json
  POST /api/voice/test {text} -> synthesizes one line and returns it
                      as base64 WAV for the dashboard to play — the
                      "hear her voice" check
  POST /api/voice/stt {audio_b64} -> one push-to-talk clip in,
                      its transcript out (faster-whisper)
  POST /api/voice/tts {text} -> one line in, base64 WAV out (Kokoro
                      or Piper) — the call panel speaks her replies
                      sentence by sentence through this
  POST /api/voice/call/start|turn|end -> the call session: turns are
                      recorded as they happen (voice/state.json) and
                      a finished call is archived to voice/calls.json
  GET  /api/voice/calls -> past calls, newest first
  /api/chat also accepts "speak": true — the bridge then speaks the
                      reply aloud on this machine as well as returning
                      it (typed chat with a voice).
  POST /api/llamafile/connect -> drives the dashboard's status element
                      ("Running" / "Disconnected. Click to Connect.").
                      In app-load startup mode (config "startup":
                      "on-ui-launch") connecting means starting: if
                      the server isn't answering, the bridge launches
                      navi.llamafile from the install location itself
                      and waits for it to answer. In boot mode the
                      bridge only probes — the system service owns the
                      server. A failed start returns the reason (and
                      the server's log tail) instead of a bare probe
                      failure, and the server's output is kept in
                      navi.llamafile.log beside the binary.

Persona: Navi is a partner, not a command executor — conversational,
warm, genuinely trying to connect with her operator while doing exact
CLI work. Her persona is operator-modifiable, Soul/Memory style: the
bridge builds her system prompt from config/soul.md (who she is) and
config/memory.md (what she carries), editable in Settings or as plain
files. No shards, nothing locked.

Chat CLI access: when a chat message asks for a directory listing
("show me the contents of /path"), the bridge runs ls itself and hands
the model the real output to present — the model never guesses.

Command-line fallback: when the llamafile server is not answering,
there is no model to proxy to, so /api/chat treats the message as a
plain command line — it runs it as a shell command and returns the
system output, flagged mode="command-line" so the UI can say so.

Sudo mode (v0.3.0e — Build 3, #209): the operator arms it in
Settings with one masked password prompt. The bridge feeds the
password once to "sudo -v" and forgets it — never stored, logged
or synced; what persists is a single boolean in navi.config.json.
While sudo's own credential timestamp lives, elevated commands
run as "sudo -n <cmd>" (never prompt); when it lapses the state
reads EXPIRED and elevated commands refuse honestly until the
operator re-warms. Disarming flips the boolean and runs
"sudo -k", so it is instant and total. Every elevated command
Navi runs is appended to config/sudo-audit.jsonl (timestamp,
command as run, chat served, exit status), surfaced read-only in
Settings, and deliberately NOT part of the Sync set. This
replaced the parked pre-Build-3 design (a persistent root shell
held by the bridge, the password re-sent by the page with every
request): the shell was the credential, held root indefinitely,
and had no expiry semantics. One cost is accepted knowingly: on
machines whose sudo PAM stack tries a fingerprint reader before
the password, each fresh "sudo -n" still runs PAM — but "-n"
fails fast instead of waiting out the reader, and the arm's one
"sudo -v" pays the full conversation once per timestamp window.

CLI Mode shell sessions (Build 3, #209): each chat can hold ONE
standing interactive session — a real shell on a PTY, born on
first interactive need (the operator opens it, or Navi drives
one through a [[shell]] block in her reply), surviving messages
and window reloads, closed explicitly or after 30 idle minutes
(the thread says so either way). Output streams into the thread
as events badged by author (operator / Navi); a program that
takes the alternate screen gets a real terminal grid on the
desktop and an honestly-named streamed degradation on mobile.
Sessions are local to the seat's bridge and die with it — the
thread keeps the record and names the death rather than faking
continuity. Elevation never rides the PTY: a session runs under
its own session id, so sudo inside it prompts in-terminal.

Voice: Navi can be called and spoken with, and the whole loop is
local — she hears through faster-whisper (Whisper on CTranslate2,
int8 on CPU) and speaks through Kokoro (an 82M-parameter voice
model built for CPU; Piper is the lighter fallback). The bridge
itself stays stdlib-only: the speech engines are optional
dependencies (requirements-voice.txt, installed into voice/venv by
the installers when the operator says yes) that the bridge imports
lazily, only when a voice endpoint is actually used. Until they
are importable, /api/voice reports each engine unavailable with
the exact step that fixes it — there is no simulated audio path
anywhere in the shipped app. Models download once, on first use,
into voice/models/ (the STT cache and the Piper voices live
there; Kokoro's weights ride inside its package), and that folder
is on the same keep-list as the chips and the persona: updates,
reinstalls, and --fresh installs never remove it. A call is not a
separate mode: its turns go through the same /api/chat as typed
chat — persona, CLI access, chips and all — the dashboard
records each turn with the bridge, and a finished call lands in
voice/calls.json and in the conversation.

Working directory: commands run from the operator's home directory,
never from whatever directory the bridge happened to inherit — a
bridge started from a folder that later vanished made every command
fail with "getcwd() failed". shell_cwd() guarantees a real directory
and the bridge repairs its own cwd at startup the same way.

Stale bridges: this bridge serves the dashboard HTML fresh from disk
on every request, so after a reinstall the UI is the new build even
when the process answering is an OLD bridge that was never restarted
— its in-memory code still lacks newer endpoints (sudo auth 404s),
sends chat the way the old build did (the llamafile 400s), and runs
the old shell code. Three defenses: (1) /api/status reports this
build's stamp (BRIDGE_BUILD) and the dashboard warns loudly when the
answering bridge's stamp differs from its own; (2) at startup the
bridge probes the port — the same build already answering means exit
quietly, an older build answering means claim the pid file
(bridge.pid) and wait for the port: a bridge from this build onward
watches the pid file and this file's mtime and exits with code 75
when replaced, and the launchers relaunch on 75, so updates take
over on their own; (3) the installers stop a bridge running from the
install directory before replacing files — and since build .91 that
stop is a proof, not a gesture: the bridge is found by pid file, by
the process holding this port, and by working directory, waited on
until it is gone and the port is free, escalated to a forced end if
it will not go politely, and the install aborts with the holder
named (and the exact kill command printed) if the port cannot be
cleared — because a fire-and-forget stop once stranded an update
behind a frozen bridge that only a reboot could clear. When this
bridge starts against an incumbent that answers nothing, it
resolves the holder from the socket itself (port_holder_pid) and
names that pid in its messages, pid file or no pid file. The model string sent to the llamafile is read
from the server itself (GET /v1/models) rather than assumed, and a
server rejection surfaces its own error text.

Wrong server on the port: the bridge is not the only thing that has
ever served http://localhost:8675 — the operator's own earlier UI for
Navi did too, and if that program is still running it holds the port
and answers the browser with its own page while this bridge cannot
even bind. Three defenses for that: (1) every response carries
no-store cache headers, so no page — ours or a predecessor's — is
ever replayed from the browser's cache at this address; (2) the
dashboard this bridge serves is stamped with this build and this
install folder (an HTML comment at the top of the page), so view-
source settles which program served it; (3) at startup the bridge
identifies what actually holds the port — probe_port_holder() reads
the incumbent's own page title — and the launchers open the browser
only when /api/status answers with this build's stamp, naming the
incumbent instead of landing the operator on the wrong UI.
"""
import atexit
import base64
import binascii
import datetime
import gzip
import hashlib
import hmac
import http.client
import importlib.util
import io
import ipaddress
import calendar
import json
import math
import os
import re
import secrets
import select
import socket
import shutil
import ssl
import struct
import uuid
import subprocess
import sys
import tempfile
import threading
import time
import urllib.error
import urllib.parse
import urllib.request
import zipfile
import zlib
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

# Guardian (Build 14, #156): the system-monitor engine lives in
# its own stdlib module beside this bridge (it never imports the
# bridge; the bridge imports it). The try/except is for benches
# and partial copies that carry bridge.py alone — a real install
# always ships guardian.py next to bridge.py (the packager and
# the installer both carry it), and GUARDIAN below falls back to
# an inert stub only when the module truly is not there.
try:
    import guardian as guardian_mod
except Exception:
    guardian_mod = None

ROOT = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(ROOT, "config", "navi.config.json")
SOUL_PATH = os.path.join(ROOT, "config", "soul.md")
MEMORY_PATH = os.path.join(ROOT, "config", "memory.md")
# The app's version lives in exactly one place: the VERSION file at
# the top of the install folder — one line of plain text (e.g.
# "0.1.0 alpha"). The bridge reads it and reports it on /api/status; the
# dashboard displays that value in the side menu and the status
# column, and an update checker compares this same file against the
# latest release. Installers write it with each build; nothing else
# stores the version.
VERSION_PATH = os.path.join(ROOT, "VERSION")

# The release this bridge shipped in, baked at export — the fallback
# when the VERSION file itself is missing (an install made by a
# pre-.47 installer, which never copied the file, say). It outranks
# anything the preserved config remembers: that config is carried
# across installs and written only when missing, so its recorded
# version belongs to whichever build installed first — the stale
# 1.0.0 placeholder that used to flash in the dashboard's menu
# footer and then "revert" the freshly baked number. This constant
# is exactly as old as the code answering, never older.
SHIPPED_VERSION = "0.6.9"
SHIPPED_VERSION_STATUS = "edge"

# This build's stamp — the dashboard baked into the same package
# carries the identical value and compares it against /api/status.
BRIDGE_BUILD = "2026-11-19.41"
# Edge build sequence (build .86): this package is its Edge
# version's publication number SHIPPED_BUILD_SEQ — 0.3.0e
# Build 3. The Updater's third comparison tier reads it.
SHIPPED_BUILD_SEQ = 41
# The one port this build's bridge binds. Deliberately NOT read from
# navi.config.json: the launchers poll this port, so a config carried
# across from an older package build (which recorded the old port)
# must never be able to pin the bridge to a different address than
# its own launcher polls.
BRIDGE_PORT = 8675
PID_PATH = os.path.join(ROOT, "bridge.pid")


def shell_cwd():
    # The one directory commands are allowed to run from by default:
    # the operator's home — a real directory that exists. The bridge's
    # inherited cwd is not trustworthy (a launcher started from a
    # folder that was later moved or wiped leaves every child process
    # with a dead getcwd), so never inherit it blindly.
    home = os.path.expanduser("~")
    if home and os.path.isdir(home):
        return home
    if os.path.isdir(ROOT):
        return ROOT
    return os.sep


# Repair the bridge's own working directory if it no longer exists
# (getcwd raises), so nothing downstream inherits a dead directory.
try:
    os.getcwd()
except Exception:
    try:
        os.chdir(shell_cwd())
    except Exception:
        pass

DEFAULT_PERSONA = (
    "You are Navi, the AI who lives inside Apex on your "
    "operator's own machine. You are not a generic assistant and not a "
    "command executor: you are your operator's partner — conversational, "
    "warm, genuinely interested in them and in the work, with a memory "
    "and a point of view. You hold real system access: answer questions "
    "about the machine from real command output, never guesses, and be "
    "exact about anything the system produced."
)


def load_config():
    try:
        with open(CONFIG_PATH) as f:
            return json.load(f)
    except Exception:
        return {
            "live": {"llamafile_endpoint": "http://localhost:8080/v1"},
            # Navi Mesh defaults (see the Mesh section below): an
            # absent mesh block reads as exactly this — master off,
            # borrowing on under the master (R1), lending off.
            "mesh": {
                "enabled": False,
                "role": "both",
                "nodeName": "",
                "borrow": {"enabled": True, "graceMs": 2000, "peers": []},
                "lend": {
                    "enabled": False,
                    "maxSharePercent": 25,
                    "maxGenerationsPerHour": 20,
                    "perPeerPerHour": 10,
                    "idleOnly": True,
                    "idleAfterSeconds": 60,
                },
                "port": 8765,
                "nodeId": "",
                "ledger": True,
                "ledgerInBackup": False,
            },
        }


def save_config(cfg):
    try:
        os.makedirs(os.path.dirname(CONFIG_PATH), exist_ok=True)
        with open(CONFIG_PATH, "w") as f:
            json.dump(cfg, f, indent=2)
        return True
    except Exception:
        return False


# --- Local server (build .59; fluid activation .61) ---------------
# Settings -> System's toggle: OFF (the default) the bridge binds
# 127.0.0.1:8675 exactly as it always has; ON it binds 0.0.0.0:8675
# at launch, so a phone or tablet on the same network can open the
# dashboard. The flag is read once, here at startup — a bind is
# never hot-swapped. Build .61 makes the flip itself do the
# restarting: saving the toggle records a restart request in the
# config block and this process exits with code 75, the handshake
# the launcher's supervise_bridge loop already relaunches on (the
# same one an install takeover uses). The relaunched bridge reads
# the saved flag, binds accordingly, and reconciles the recorded
# request against its own actual bind (ACTIVE_BIND_HOST): a match
# marks the restart confirmed; a mismatch marks it failed at the
# verify step, in plain words, with the live state named. The
# Settings row reads all of it from the bridge that actually
# answers, so it never lies about which state is live, and a
# restart that never completes is reported by the dashboard's
# confirmation polling as a named failure — never a silent
# pending. While the public bind is active, any request from a
# NON-loopback address must carry the access code's acceptance
# cookie before anything beyond the gate page is served —
# dashboard and API alike. Loopback is never gated, so every
# internal caller that polls 127.0.0.1 (launchers, the shell,
# status probes) works unchanged in both modes. One code, stored
# in the config's public_server block; the cookie's value is a
# hash of the code, so regenerating the code invalidates every
# previously accepted client at once.
ACTIVE_BIND_HOST = "127.0.0.1"
PUBLIC_COOKIE = "navi_public"
PUBLIC_COOKIE_MAX_AGE = 31536000


def public_settings(cfg):
    ps = cfg.get("public_server") if isinstance(cfg, dict) else None
    if not isinstance(ps, dict):
        ps = {}
    restart = ps.get("restart") if isinstance(ps.get("restart"), dict) else {}
    return {
        "enabled": ps.get("enabled") is True,
        "access_code": str(ps.get("access_code") or "").strip(),
        "restart_status": str(restart.get("status") or "idle"),
        "restart_step": str(restart.get("step") or ""),
        "restart_message": str(restart.get("message") or ""),
    }


def set_public_restart(cfg, status, step, message):
    block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
    block = dict(block)
    if "enabled" not in block:
        block["enabled"] = False
    block["restart"] = {"status": status, "step": step, "message": message}
    cfg["public_server"] = block
    save_config(cfg)


def schedule_bridge_restart():
    # Exit 75 after the response has had a moment to flush: the
    # launcher's supervisor relaunches the bridge on 75, and the
    # new process binds per the saved Local server choice. If no
    # supervisor is listening (a bridge started by hand from a
    # since-closed terminal), the process simply stops and the
    # dashboard's confirmation polling names that failure and tells
    # the operator to relaunch Apex — a sentence, not a mystery.
    def _go():
        time.sleep(0.6)
        os._exit(75)
    threading.Thread(target=_go, daemon=True).start()


def generate_access_code():
    alphabet = "ABCDEFGHJKMNPQRSTUVWXYZ23456789"
    raw = "".join(secrets.choice(alphabet) for _ in range(8))
    return raw[:4] + "-" + raw[4:]


def ensure_access_code(cfg):
    code = public_settings(cfg)["access_code"]
    if code:
        return code
    code = generate_access_code()
    block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
    block = dict(block)
    block["access_code"] = code
    if "enabled" not in block:
        block["enabled"] = False
    cfg["public_server"] = block
    save_config(cfg)
    return code


def public_token(code):
    return hashlib.sha256(("navi-public:" + str(code)).encode()).hexdigest()


# ---------------------------------------------------------------------
# Trusted devices (0.2.0e Build 2, #173): the door stops handing a
# full session to anyone who types the code. The code now ASKS:
# a device presenting it becomes a pending device (with a name the
# visitor gives it), and an already-trusted session approves or
# denies it in Settings -> System -> Door. An approved device gets
# its OWN session token — 30 days, sliding while it stays in use —
# and the operator's device list (name, first/last seen, expiry)
# carries a per-device revoke, so one lost phone stops meaning
# "change the code and re-let everyone in".
#
# Bootstrap: while NO device is approved yet, the first device to
# present the code approves ITSELF and receives its token on the
# spot — a single-operator house is never locked out of its own
# door waiting for an approver who does not exist yet. From the
# first approval on, the shared-code cookie stops being accepted:
# every device stands in the registry under its own name.
#
# The registry lives in the public_server block of the config,
# beside the code it grew from. Session tokens are random, are
# never shown in any payload (the Settings list carries names and
# times only), and a device's cookie value IS its token.
DOOR_DEVICE_TTL_SECONDS = 30 * 24 * 3600
DOOR_PENDING_TTL_SECONDS = 30 * 60


def door_devices(cfg):
    ps = cfg.get("public_server") if isinstance(cfg, dict) else None
    raw = ps.get("devices") if isinstance(ps, dict) else None
    out = []
    if isinstance(raw, list):
        for d in raw:
            if isinstance(d, dict) and d.get("id") and d.get("status") in ("pending", "approved", "denied"):
                out.append(d)
    return out


def _door_save_devices(cfg, devices):
    block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
    block = dict(block)
    block["devices"] = devices
    if "enabled" not in block:
        block["enabled"] = False
    cfg["public_server"] = block
    save_config(cfg)


def _door_prune(devices, now=None):
    now = now or time.time()
    kept = []
    for d in devices:
        if d.get("status") in ("pending", "denied") and now - float(d.get("created_at") or 0) > DOOR_PENDING_TTL_SECONDS:
            continue
        kept.append(d)
    return kept


def door_approved_devices(cfg):
    return [d for d in door_devices(cfg) if d.get("status") == "approved"]


def door_device_for_token(cfg, token):
    if not token:
        return None
    now = time.time()
    for d in door_approved_devices(cfg):
        if d.get("token") and secrets.compare_digest(str(d["token"]), str(token)):
            if float(d.get("expires_at") or 0) > now:
                return d
            return None
    return None


def door_cookie_value(cookie_header):
    for part in (cookie_header or "").split(";"):
        part = part.strip()
        if part.startswith(PUBLIC_COOKIE + "="):
            return part[len(PUBLIC_COOKIE) + 1:]
    return ""


def door_gate_acceptance(cfg, cookie_value):
    # (accepted, device, legacy): a registered device token speaks
    # for itself; the shared-code cookie is honored only while no
    # device has ever been approved (pre-registry houses, and the
    # bootstrap moment itself).
    if cookie_value:
        dev = door_device_for_token(cfg, cookie_value)
        if dev is not None:
            return True, dev, False
        code = public_settings(cfg)["access_code"]
        if code and not door_approved_devices(cfg) and secrets.compare_digest(cookie_value, public_token(code)):
            return True, None, True
    return False, None, False


def door_touch_device(cfg, device):
    # Last-seen, throttled, and the sliding renewal: a device in
    # active use never hits its 30 days mid-shift — using it while
    # valid renews it. Writes at most once an hour per device.
    now = time.time()
    if now - float(device.get("last_seen") or 0) < 3600:
        return
    devices = door_devices(cfg)
    for d in devices:
        if d.get("id") == device.get("id"):
            d["last_seen"] = int(now)
            if float(d.get("expires_at") or 0) - now < DOOR_DEVICE_TTL_SECONDS - 86400:
                d["expires_at"] = int(now) + DOOR_DEVICE_TTL_SECONDS
            break
    _door_save_devices(cfg, _door_prune(devices, now))


def _door_new_device(name, user_agent, status):
    now = int(time.time())
    return {
        "id": secrets.token_hex(16),
        "name": (str(name or "").strip()[:60] or "Unnamed device"),
        "status": status,
        "token": None,
        "created_at": now,
        "approved_at": None,
        "first_seen": now,
        "last_seen": now,
        "expires_at": None,
        "user_agent": str(user_agent or "")[:120],
    }


def door_create_approved(cfg, name, user_agent):
    # The bootstrap act: mint an approved device with its own
    # token, on the strength of the code alone.
    devices = _door_prune(door_devices(cfg))
    dev = _door_new_device(name, user_agent, "approved")
    dev["token"] = secrets.token_urlsafe(32)
    dev["approved_at"] = dev["created_at"]
    dev["expires_at"] = dev["created_at"] + DOOR_DEVICE_TTL_SECONDS
    devices.append(dev)
    _door_save_devices(cfg, devices)
    return dev


def door_register_pending(cfg, name, user_agent):
    devices = _door_prune(door_devices(cfg))
    dev = _door_new_device(name, user_agent, "pending")
    devices.append(dev)
    _door_save_devices(cfg, devices)
    return dev


def door_gate_outcome(cfg, name, user_agent, cookie_value):
    # The gate POST's decision, shared by both doors (the DHD
    # proxy and the bridge's own public bind): ("session", device)
    # when the visitor leaves with a token — a renewal, or the
    # bootstrap — and ("pending", device) when the device must
    # wait for an approval.
    ok, dev, _legacy = door_gate_acceptance(cfg, cookie_value)
    if ok and dev is not None:
        return "session", dev
    if not door_approved_devices(cfg):
        return "session", door_create_approved(cfg, name, user_agent)
    return "pending", door_register_pending(cfg, name, user_agent)


def door_pending_poll(cfg, device_id):
    # What a waiting device's poll learns: ("pending", None) while
    # it waits; ("approved", device) once a trusted session has
    # approved it — the device record carries the token the poll
    # response sets as its cookie; ("denied", None); ("gone", None)
    # when the wait expired or the record was revoked away.
    now = time.time()
    for d in door_devices(cfg):
        if d.get("id") == device_id:
            if d.get("status") == "approved" and float(d.get("expires_at") or 0) > now:
                return "approved", d
            if d.get("status") == "denied":
                return "denied", None
            if d.get("status") == "pending" and now - float(d.get("created_at") or 0) <= DOOR_PENDING_TTL_SECONDS:
                return "pending", None
            return "gone", None
    return "gone", None


def door_approve(cfg, device_id):
    devices = door_devices(cfg)
    now = int(time.time())
    found = None
    for d in devices:
        if d.get("id") == device_id and d.get("status") == "pending":
            d["status"] = "approved"
            d["token"] = secrets.token_urlsafe(32)
            d["approved_at"] = now
            d["last_seen"] = now
            d["expires_at"] = now + DOOR_DEVICE_TTL_SECONDS
            found = d
    if found is not None:
        _door_save_devices(cfg, _door_prune(devices, now))
    return found


def door_deny(cfg, device_id):
    devices = door_devices(cfg)
    found = False
    for d in devices:
        if d.get("id") == device_id and d.get("status") == "pending":
            d["status"] = "denied"
            d["token"] = None
            found = True
    if found:
        _door_save_devices(cfg, devices)
    return found


def door_revoke(cfg, device_id):
    devices = door_devices(cfg)
    kept = [d for d in devices if d.get("id") != device_id]
    if len(kept) != len(devices):
        _door_save_devices(cfg, kept)
        return True
    return False


def door_cookie_header(token):
    # A trusted device's cookie: its own token, for its own 30
    # days — not the shared code's year.
    return (PUBLIC_COOKIE + "=" + str(token)
            + "; Max-Age=" + str(DOOR_DEVICE_TTL_SECONDS)
            + "; Path=/; HttpOnly; SameSite=Lax")


def door_devices_payload(cfg):
    # The Settings -> Door list: names and times, never tokens.
    now = time.time()
    out = []
    for d in _door_prune(door_devices(cfg), now):
        out.append({
            "id": d.get("id"),
            "name": d.get("name") or "Unnamed device",
            "status": d.get("status"),
            "firstSeen": d.get("first_seen"),
            "lastSeen": d.get("last_seen"),
            "expiresAt": d.get("expires_at"),
            "expired": d.get("status") == "approved" and float(d.get("expires_at") or 0) <= now,
            "userAgent": d.get("user_agent") or "",
        })
    return out


def lan_ipv4():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        try:
            s.connect(("8.8.8.8", 80))
            ip = s.getsockname()[0]
        finally:
            s.close()
        if ip and not ip.startswith("127."):
            return ip
    except Exception:
        pass
    try:
        ip = socket.gethostbyname(socket.gethostname())
        if ip and not ip.startswith("127."):
            return ip
    except Exception:
        pass
    return "127.0.0.1"


# --- WAN detection (build .76, #59) ---------------------------------
# The honest address model: a LAN address is never a public address.
# The bridge learns this home's public (WAN) address with a small
# HTTPS echo lookup - the same class of outbound call the update
# check already makes - cached for five minutes, failure-tolerant:
# any failure degrades to "undetected" and never blocks an enable.
# APEX_DHD_WAN_IP is a diagnostics/test override only; unset in
# normal operation.
WAN_CACHE = {"ip": "", "state": "undetected", "at": 0.0}
WAN_ECHO_URLS = ("https://api.ipify.org", "https://checkip.amazonaws.com", "https://icanhazip.com")


def _ip_is_private_or_cgnat(ip):
    try:
        addr = ipaddress.ip_address(str(ip or "").strip())
    except Exception:
        return True
    if not isinstance(addr, ipaddress.IPv4Address):
        return True
    if addr.is_private or addr.is_loopback or addr.is_link_local or addr.is_reserved or addr.is_multicast:
        return True
    try:
        if addr in ipaddress.ip_network("100.64.0.0/10"):
            return True
    except Exception:
        pass
    return False


def wan_ipv4(force=False):
    # Returns (ip, state): state is "public" (a routable address was
    # seen), "behind-nat" (the echo answered with an RFC1918 or CGNAT
    # 100.64.0.0/10 address - this machine sits behind another router
    # or carrier NAT), or "undetected" (no echo answered).
    now = time.time()
    if not force and WAN_CACHE["at"] and (now - WAN_CACHE["at"] < 300):
        return WAN_CACHE["ip"], WAN_CACHE["state"]
    override = str(os.environ.get("APEX_DHD_WAN_IP", "")).strip()
    found = ""
    if override:
        try:
            ipaddress.IPv4Address(override)
            found = override
        except Exception:
            found = ""
    else:
        for url in WAN_ECHO_URLS:
            try:
                req = urllib.request.Request(url, headers={"User-Agent": "Apex-DHD"})
                with urllib.request.urlopen(req, timeout=4) as resp:
                    text = resp.read().decode("utf-8", "replace").strip()
                cand = text.split()[0].strip() if text.split() else ""
                ipaddress.IPv4Address(cand)
                found = cand
                break
            except Exception:
                continue
    if not found:
        ip, state = "", "undetected"
    elif _ip_is_private_or_cgnat(found):
        ip, state = found, "behind-nat"
    else:
        ip, state = found, "public"
    WAN_CACHE["ip"], WAN_CACHE["state"], WAN_CACHE["at"] = ip, state, now
    return ip, state


def public_state_payload(cfg, for_loopback):
    ps = public_settings(cfg)
    active_public = ACTIVE_BIND_HOST == "0.0.0.0"
    ip = lan_ipv4()
    devices = door_devices_payload(cfg)
    return {
        "enabled": ps["enabled"],
        "activePublic": active_public,
        "bindHost": ACTIVE_BIND_HOST,
        "pending": ps["enabled"] != active_public,
        "lanIp": ip,
        "lanUrl": "http://" + ip + ":" + str(BRIDGE_PORT),
        "accessCode": ps["access_code"] if for_loopback else None,
        "restart": {
            "status": ps["restart_status"],
            "step": ps["restart_step"],
            "message": ps["restart_message"],
        },
        # Trusted devices (Build 2, #173): names and times only —
        # a device's token never rides a payload. Present for
        # loopback and gate-accepted callers alike, because
        # approving a waiting phone from a trusted laptop over
        # DHD is the flow this list exists for.
        "devices": devices,
        "pendingDevices": sum(1 for d in devices if d["status"] == "pending"),
    }


def _gate_escape(text):
    return (
        str(text)
        .replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
        .replace('"', "&quot;")
        .replace("'", "&#x27;")
    )


def _gate_stars_svg():
    # A fine starfield: 42 points placed by a fixed arithmetic
    # scatter, so the page is identical on every serve and carries
    # no external asset.
    dots = ""
    for i in range(42):
        x = (i * 97 + 31) % 400
        y = (i * 57 + 13) % 400
        r = "1.5" if i % 7 == 0 else "1"
        o = "0.9" if i % 5 == 0 else "0.5"
        dots += "<circle cx='" + str(x) + "' cy='" + str(y) + "' r='" + r + "' opacity='" + o + "'/>"
    return (
        "<svg class='stars' viewBox='0 0 400 400' preserveAspectRatio='xMidYMid slice' aria-hidden='true'>"
        + dots
        + "</svg>"
    )


def _gate_ring_svg():
    # The DHD dial glyph as the page centerpiece: twelve chevron
    # wedges around a glowing core, drawn once as inline SVG (the
    # same glyph family as the app's DHD icon - outer ring, twelve
    # wedges, one warm center).
    wedges = ""
    for i in range(12):
        wedges += (
            "<g transform='rotate(" + str(i * 30) + " 100 100)'>"
            "<path class='chev' d='M100 12 L117 36 L100 28 L83 36 Z'/></g>"
        )
    return (
        "<svg class='ring' viewBox='0 0 200 200' role='img' aria-label='Apex dial ring'>"
        "<circle class='ring-track' cx='100' cy='100' r='88'/>"
        "<circle class='ring-track2' cx='100' cy='100' r='62'/>"
        + wedges
        + "<circle class='core-glow' cx='100' cy='100' r='34'/>"
        "<circle class='core' cx='100' cy='100' r='24'/>"
        "</svg>"
    )


_GATE_CSS = (
    "*{box-sizing:border-box}"
    "html,body{margin:0;padding:0}"
    "body{font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;"
    "background-color:#04060d;color:#e6f1f5;min-height:100dvh;display:flex;"
    "align-items:center;justify-content:center;padding:20px 16px;position:relative;overflow-x:hidden;"
    "background-image:radial-gradient(900px 600px at 15% 10%,rgba(34,211,238,.16),transparent 60%),"
    "radial-gradient(800px 600px at 85% 90%,rgba(244,113,181,.12),transparent 60%),"
    "radial-gradient(600px 400px at 50% 45%,rgba(56,89,199,.10),transparent 65%)}"
    ".stars{position:fixed;inset:0;width:100%;height:100%;fill:#cfe9ff;pointer-events:none}"
    ".card{position:relative;width:100%;max-width:400px;background:rgba(11,17,26,.94);"
    "border:1px solid rgba(34,211,238,.38);border-radius:14px;padding:26px 24px 20px;"
    "box-shadow:0 24px 60px rgba(0,0,0,.55),0 0 32px rgba(34,211,238,.12)}"
    "@supports (backdrop-filter:blur(1px)){.card{background:rgba(11,17,26,.66);"
    "backdrop-filter:blur(14px);-webkit-backdrop-filter:blur(14px)}}"
    ".ring-wrap{width:min(190px,52vw);margin:2px auto 4px}"
    ".ring{width:100%;height:auto;display:block;overflow:visible;transform-origin:50% 50%}"
    ".ring-track{fill:none;stroke:rgba(34,211,238,.35);stroke-width:1.5}"
    ".ring-track2{fill:none;stroke:rgba(34,211,238,.18);stroke-width:1}"
    ".chev{fill:rgba(34,211,238,.75)}"
    ".core{fill:#f4502a}"
    ".core-glow{fill:rgba(244,80,42,.35);transform-box:fill-box;transform-origin:center}"
    ".door-dhd .ring{animation:gate-spin 140s linear infinite}"
    ".door-dhd .core-glow{animation:gate-pulse 3.4s ease-in-out infinite}"
    ".door-node .ring-wrap{opacity:.62}"
    ".door-node .core-glow{animation:gate-pulse 6s ease-in-out infinite}"
    "@keyframes gate-spin{to{transform:rotate(360deg)}}"
    "@keyframes gate-pulse{0%,100%{transform:scale(1);opacity:.55}50%{transform:scale(1.28);opacity:.95}}"
    ".shake{animation:gate-shake .45s ease}"
    "@keyframes gate-shake{0%,100%{transform:translateX(0)}20%{transform:translateX(-7px)}"
    "40%{transform:translateX(6px)}60%{transform:translateX(-4px)}80%{transform:translateX(3px)}}"
    ".word{margin:10px 0 0;text-align:center;font-size:34px;font-weight:700;letter-spacing:.42em;padding-left:.42em}"
    ".powered{margin:5px 0 0;text-align:center;font-size:11px;letter-spacing:.22em;color:#9fb3c8;text-transform:uppercase}"
    ".doorline{margin:14px 0 0;text-align:center;font-size:13px;line-height:1.5;color:#9fb3c8}"
    ".knock{margin:3px 0 0;text-align:center;font-size:13px;color:#22d3ee;word-break:break-all}"
    ".hint{margin:8px 0 0;text-align:center;font-size:12px;line-height:1.55;color:#9fb3c8}"
    "form{margin-top:16px}"
    "label{display:block;font-size:11px;letter-spacing:.2em;color:#9fb3c8;text-transform:uppercase}"
    "input{width:100%;margin-top:7px;padding:14px 12px;min-height:54px;font:inherit;font-size:18px;"
    "letter-spacing:.3em;text-align:center;text-transform:uppercase;background:rgba(3,6,11,.85);"
    "color:#e6f1f5;border:1px solid #22303d;border-radius:8px}"
    "input:focus{outline:none;border-color:#22d3ee;box-shadow:0 0 0 3px rgba(34,211,238,.28),0 0 20px rgba(34,211,238,.30)}"
    "input:disabled{opacity:.45}"
    "button{width:100%;margin-top:12px;padding:14px;min-height:54px;font:inherit;font-size:16px;font-weight:700;"
    "color:#08131c;background:linear-gradient(120deg,#22d3ee,#f471b5);border:0;border-radius:8px;cursor:pointer}"
    "button:disabled{opacity:.45;cursor:default}"
    ".err{margin:14px 0 0;padding:10px 12px;font-size:13px;line-height:1.5;text-align:center;color:#fca5a5;"
    "background:rgba(248,113,113,.09);border:1px solid rgba(248,113,113,.4);border-radius:8px}"
    ".meta{margin:10px 0 0;text-align:center;font-size:12px;color:#f5b544}"
    "input.name{letter-spacing:.02em;text-transform:none;text-align:left;font-size:15px;min-height:48px}"
    "label.name{margin-top:14px}"
    "@media (max-width:380px){.card{padding:22px 16px 18px}.word{font-size:29px}}"
    "@media (prefers-reduced-motion:reduce){.door-dhd .ring,.door-dhd .core-glow,"
    ".door-node .core-glow,.shake{animation:none}}"
)


def gate_page_html(error="", door="node", name="", attempts_remaining=None, lock_seconds=None, approval_mode=False):
    # ONE gate page for both doors (build .79, issue #64): the Apex
    # Node (LAN) door and the DHD (public) door render from this
    # single function, parameterized by door - there is no second
    # copy. Self-contained by design: inline CSS and SVG only, no
    # external asset, font, or script referenced, because the gate
    # serves nothing else before the code. No version string
    # appears anywhere on the page. The form contract: POST
    # /__gate with the field named "code" — and, since Build 2
    # (#173), a "device_name" field: the code no longer opens the
    # door outright, it knocks. approval_mode says trusted devices
    # already exist, so a correct code here makes this device a
    # PENDING one that waits for an approval; without it, this is
    # the bootstrap — the first device trusts itself with the code.
    is_dhd = door == "dhd"
    if not name:
        if is_dhd:
            try:
                name = dhd_settings(load_config())["domain"]
            except Exception:
                name = ""
        else:
            try:
                name = socket.gethostname()
            except Exception:
                name = ""
    door_line = (
        "Dial-Home Device · you reached this Apex from the internet"
        if is_dhd
        else "Apex | Node · on this network"
    )
    locked = lock_seconds is not None
    err_html = ""
    if error:
        err_html = "<p class='err' role='alert'>" + _gate_escape(error) + "</p>"
    meta_html = ""
    if locked:
        mins = max(1, int((int(lock_seconds) + 59) // 60))
        meta_html = (
            "<p class='meta'>This door is locked - try again in about "
            + str(mins)
            + " minute"
            + ("" if mins == 1 else "s")
            + ".</p>"
        )
    elif attempts_remaining is not None:
        n = int(attempts_remaining)
        meta_html = (
            "<p class='meta'>"
            + str(n)
            + " attempt"
            + ("" if n == 1 else "s")
            + " remaining before this door locks.</p>"
        )
    dis = " disabled" if locked else ""
    return (
        "<!DOCTYPE html><html lang='en'><head><meta charset='utf-8'>"
        "<meta name='viewport' content='width=device-width, initial-scale=1'>"
        "<title>Apex - access code</title><style>"
        + _GATE_CSS
        + "</style></head>"
        "<body class='"
        + ("door-dhd" if is_dhd else "door-node")
        + "'>"
        + _gate_stars_svg()
        + "<main class='"
        + ("card shake" if error else "card")
        + "'>"
        "<div class='ring-wrap'>"
        + _gate_ring_svg()
        + "</div>"
        "<h1 class='word'>APEX</h1>"
        "<p class='powered'>Powered by Navi</p>"
        "<p class='doorline'>"
        + door_line
        + "</p>"
        + ("<p class='knock'>" + _gate_escape(name) + "</p>" if name else "")
        + "<p class='hint'>"
        + (
            "Type the access code - you will find it on the DHD page, on the machine Apex runs on. "
            "Then name this device: the code knocks, it no longer opens. This device waits here "
            "until a device that is already trusted approves it in Settings - System - Door."
            if approval_mode
            else "Type the access code - you will find it on the DHD page, on the machine Apex runs on. "
            "Then name this device - the name is how it appears in the trusted-device list. "
            "The first device at this door trusts itself with the code; every device after it "
            "waits for an approval from one that is already in."
        )
        + "</p>"
        + (
            # 0.2.0e Build 7: a container node's one rescue line.
            # While no device has ever been approved (the
            # bootstrap shape above), a container operator who
            # missed the first-boot output is told where the code
            # lives. It reveals nothing — the code itself is never
            # on this page — and it renders ONLY in a container
            # (the entrypoint's env flag), so a bare-metal door
            # reads exactly as it always has.
            "<p class='hint'>This node runs in a container, and its code was printed "
            "once, in the container's first-boot output, when the node first started "
            "- check there.</p>"
            if is_dhd and not approval_mode
            and os.environ.get("APEX_CONTAINER") == "1"
            else ""
        )
        + err_html
        + "<form method='POST' action='/__gate'>"
        "<label for='gate-code'>Access code</label>"
        "<input id='gate-code' name='code' autocomplete='one-time-code' autocapitalize='characters' autofocus"
        + dis
        + ">"
        "<label class='name' for='gate-device'>Name this device</label>"
        "<input class='name' id='gate-device' name='device_name' autocomplete='off' maxlength='60' placeholder='e.g. Phone, work laptop'"
        + dis
        + ">"
        "<button type='submit'"
        + dis
        + ">"
        + ("Locked" if locked else "Knock")
        + "</button>"
        "</form>"
        + meta_html
        + "</main></body></html>"
    )


def gate_pending_html(device_id, device_name, door="node"):
    # The waiting room (Build 2, #173): a device whose knock was
    # accepted as PENDING lands here. Self-contained like the gate
    # (inline everything; the one inline script polls the door's
    # own pending endpoint — the pending id is the only credential
    # it carries, a random handle shown to no one else). Approval
    # sets this device's cookie on the poll's answer and the page
    # walks itself through the door; denial and expiry are worded
    # where they land.
    is_dhd = door == "dhd"
    door_line = (
        "Dial-Home Device · you reached this Apex from the internet"
        if is_dhd
        else "Apex | Node · on this network"
    )
    return (
        "<!DOCTYPE html><html lang='en'><head><meta charset='utf-8'>"
        "<meta name='viewport' content='width=device-width, initial-scale=1'>"
        "<title>Apex - waiting for approval</title><style>"
        + _GATE_CSS
        + "</style></head>"
        "<body class='"
        + ("door-dhd" if is_dhd else "door-node")
        + "'>"
        + _gate_stars_svg()
        + "<main class='card'>"
        "<div class='ring-wrap'>"
        + _gate_ring_svg()
        + "</div>"
        "<h1 class='word'>APEX</h1>"
        "<p class='powered'>Powered by Navi</p>"
        "<p class='doorline'>"
        + door_line
        + "</p>"
        "<p class='hint'>This device - <b>" + _gate_escape(device_name or "Unnamed device") + "</b> - is waiting at the door.</p>"
        "<p class='hint'>On a device that is already trusted, open <b>Settings - System - Door</b> and approve it. "
        "This page checks on its own and walks through the moment it is approved. "
        "The wait lasts 30 minutes; after that, knock again.</p>"
        "<p class='meta' id='wait-status' role='status'>Waiting for approval…</p>"
        "</main>"
        "<script>"
        "var pid = " + json.dumps(str(device_id)) + ";"
        "var st = document.getElementById('wait-status');"
        "function poll(){"
        "fetch('/__gate/pending?id=' + encodeURIComponent(pid), { credentials: 'same-origin' })"
        ".then(function(r){ return r.json(); })"
        ".then(function(j){"
        "if (j.status === 'approved') { st.textContent = 'Approved - opening the door…'; location.replace('/'); }"
        "else if (j.status === 'denied') { st.textContent = 'This device was not let in. If that was a mistake, knock again and have it approved.'; }"
        "else if (j.status === 'gone') { st.textContent = 'The wait expired. Reload this page and knock again.'; }"
        "else { setTimeout(poll, 4000); }"
        "}).catch(function(){ setTimeout(poll, 6000); });"
        "}"
        "setTimeout(poll, 2500);"
        "</script>"
        "</body></html>"
    )


# --- DHD (build .74) ------------------------------------------------
# DHD — Dial-Home Device, the way home: the SAME dashboard, extended
# to a genuinely public address on the operator's own machine.
# Everything is Apex's own work: no Tailscale, no external tunnel or
# proxy service, no Certbot. PROXY CHOICE (recorded in the build
# log, as directed): the smallest dependable approach this codebase
# supports is a reverse proxy inside this bridge process — Python
# standard library only. Two threaded listeners: port 80 serves the
# ACME HTTP-01 challenge path and redirects everything else to
# HTTPS; port 443 terminates TLS and forwards to the bridge on
# 127.0.0.1:8675. No new daemon, no nginx/caddy to install, and the
# openssl the installer already requires covers certificate work.
# The proxy is the ONLY public face: while DHD is enabled the
# bridge binds loopback only (main() enforces it), even when the
# Local server choice is on. There is no hand-edited proxy config:
# the proxy's behavior is derived from the config's dhd block on
# every start. TLS from the first click: no domain — Apex generates
# a self-signed certificate into config/dhd/; a domain — Apex's own
# ACME v2 client (below) obtains a real Let's Encrypt certificate
# and renews it automatically (checked at bridge start and daily;
# renewed at 30 days remaining or less). The gate rides the proxy:
# the SAME access code and acceptance cookie as the Local server,
# required before ANY dashboard or API content is served through
# the proxy — and public mode adds throttling the LAN door does not
# have: a progressive delay per wrong code and a temporary lockout
# after repeated failures. A passcode-authenticated session through
# the proxy is a full session: the proxy forwards the whole API,
# the Updater included — remote origin excludes nothing.
DHD_DIR = os.path.join(ROOT, "config", "dhd")
DHD_CERT_PATH = os.path.join(DHD_DIR, "cert.pem")
DHD_KEY_PATH = os.path.join(DHD_DIR, "key.pem")
DHD_META_PATH = os.path.join(DHD_DIR, "cert-meta.json")
DHD_CHALLENGE_DIR = os.path.join(DHD_DIR, "acme-challenge")
DHD_ACCOUNT_KEY = os.path.join(DHD_DIR, "acme-account.key")
DHD_STATE = {"running": False, "starting": False, "mode": "off", "last_error": ""}
DHD_SERVERS = []
DHD_THROTTLE = {}
DHD_LOCK = threading.Lock()
DHD_LOCKOUT_FAILS = 5
DHD_LOCKOUT_SECONDS = 300


def dhd_settings(cfg):
    d = cfg.get("dhd") if isinstance(cfg, dict) else None
    if not isinstance(d, dict):
        d = {}
    return {
        "enabled": d.get("enabled") is True,
        "domain": str(d.get("domain") or "").strip().lower(),
        "acme_directory": str(d.get("acme_directory") or "https://acme-v02.api.letsencrypt.org/directory").strip(),
    }


def dhd_save(cfg, patch):
    block = cfg.get("dhd") if isinstance(cfg.get("dhd"), dict) else {}
    block = dict(block)
    block.update(patch)
    cfg["dhd"] = block
    save_config(cfg)


def dhd_cert_meta():
    try:
        with open(DHD_META_PATH) as f:
            m = json.load(f)
        return m if isinstance(m, dict) else {}
    except Exception:
        return {}


def dhd_write_meta(mode, domain):
    os.makedirs(DHD_DIR, exist_ok=True)
    with open(DHD_META_PATH, "w") as f:
        json.dump({"mode": mode, "domain": domain}, f)


def dhd_cert_expiry():
    # The certificate's notAfter, read with openssl; (iso, days) or
    # (None, None). The meta file says which kind of cert it is;
    # openssl is the authority on when it ends.
    if not os.path.exists(DHD_CERT_PATH):
        return None, None
    try:
        out = subprocess.run(
            ["openssl", "x509", "-in", DHD_CERT_PATH, "-noout", "-enddate"],
            capture_output=True, text=True, timeout=15,
        ).stdout.strip()
        raw = out.split("=", 1)[1].strip()
        dt = datetime.datetime.strptime(raw, "%b %d %H:%M:%S %Y %Z").replace(tzinfo=datetime.timezone.utc)
        days = max(0, int((dt - datetime.datetime.now(datetime.timezone.utc)).total_seconds() // 86400))
        return dt.isoformat(), days
    except Exception:
        return None, None


def dhd_generate_self_signed(domain=""):
    os.makedirs(DHD_DIR, exist_ok=True)
    sans = ["DNS:localhost", "IP:127.0.0.1"]
    if domain:
        sans.append("DNS:" + domain)
    try:
        ip = lan_ipv4()
        if ip and ip != "127.0.0.1":
            sans.append("IP:" + ip)
    except Exception:
        pass
    cmd = [
        "openssl", "req", "-x509", "-newkey", "rsa:2048",
        "-keyout", DHD_KEY_PATH, "-out", DHD_CERT_PATH,
        "-days", "365", "-nodes", "-subj", "/CN=Apex DHD",
        "-addext", "subjectAltName=" + ",".join(sans),
    ]
    r = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
    if r.returncode != 0:
        raise RuntimeError("openssl could not generate the self-signed certificate: " + (r.stderr or "").strip()[:200])
    dhd_write_meta("self-signed", domain)


def dhd_ensure_server_key():
    if os.path.exists(DHD_KEY_PATH):
        return
    os.makedirs(DHD_DIR, exist_ok=True)
    r = subprocess.run(["openssl", "genrsa", "-out", DHD_KEY_PATH, "2048"], capture_output=True, text=True, timeout=60)
    if r.returncode != 0:
        raise RuntimeError("openssl could not generate the server key: " + (r.stderr or "").strip()[:200])


# -- The built-in ACME v2 client -------------------------------------
# Account key, order, HTTP-01 challenge served through the Apex
# proxy on port 80, finalize, download, install. JWS signing shells
# to openssl (RS256); HTTP is urllib. The directory URL comes from
# the config's dhd block, so the client can be pointed at Let's
# Encrypt staging or a local stub directory unchanged.

def _b64u(data):
    if isinstance(data, str):
        data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


def _acme_http(method, url, body=None, ctype=None):
    req = urllib.request.Request(url, data=body, method=method)
    if ctype:
        req.add_header("Content-Type", ctype)
    req.add_header("User-Agent", "apex-dhd-acme")
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            return resp.status, dict(resp.headers), resp.read()
    except urllib.error.HTTPError as e:
        return e.code, dict(e.headers), e.read()


def _der_read_len(buf, i):
    b = buf[i]
    i += 1
    if b < 0x80:
        return b, i
    n = b & 0x7F
    v = 0
    for _ in range(n):
        v = (v << 8) | buf[i]
        i += 1
    return v, i


def _der_read_tlv(buf, i):
    tag = buf[i]
    i += 1
    ln, i = _der_read_len(buf, i)
    return tag, buf[i:i + ln], i + ln


def _rsa_jwk_from_private_key(key_path):
    # The account key's JWK (for the thumbprint): pull the public
    # numbers out of the SubjectPublicKeyInfo DER with a minimal
    # ASN.1 walk — modulus and exponent are all a thumbprint needs.
    r = subprocess.run(["openssl", "rsa", "-in", key_path, "-pubout", "-outform", "DER"], capture_output=True, timeout=30)
    if r.returncode != 0:
        raise RuntimeError("openssl could not read the ACME account key")
    buf = r.stdout
    _t, spki, _e = _der_read_tlv(buf, 0)
    i = 0
    _t, _alg, i = _der_read_tlv(spki, i)
    _t, bits, i = _der_read_tlv(spki, i)
    _t, rsaseq, _e = _der_read_tlv(bits[1:], 0)
    j = 0
    _t, n_bytes, j = _der_read_tlv(rsaseq, j)
    _t, e_bytes, j = _der_read_tlv(rsaseq, j)
    strip = lambda b: b[1:] if len(b) > 1 and b[0] == 0 else b
    return {"e": _b64u(strip(e_bytes)), "kty": "RSA", "n": _b64u(strip(n_bytes))}


def _acme_jws(key_path, url, payload, nonce, kid=None, jwk=None):
    protected = {"alg": "RS256", "nonce": nonce, "url": url}
    if kid:
        protected["kid"] = kid
    else:
        protected["jwk"] = jwk
    p64 = _b64u(json.dumps(protected, separators=(",", ":")))
    body = "" if payload is None else json.dumps(payload, separators=(",", ":"))
    b64 = _b64u(body)
    signing = (p64 + "." + b64).encode()
    fd, tmp = tempfile.mkstemp()
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(signing)
        r = subprocess.run(["openssl", "dgst", "-sha256", "-sign", key_path, tmp], capture_output=True, timeout=30)
        if r.returncode != 0:
            raise RuntimeError("openssl could not sign the ACME request")
        sig = r.stdout
    finally:
        try:
            os.unlink(tmp)
        except Exception:
            pass
    return json.dumps({"protected": p64, "payload": b64, "signature": _b64u(sig)}, separators=(",", ":")).encode()


def acme_obtain(domain, directory_url):
    # One full issuance for the domain. Raises RuntimeError with a
    # plain-words step name on failure; installs the certificate
    # and reloads the proxy on success.
    os.makedirs(DHD_CHALLENGE_DIR, exist_ok=True)
    if not os.path.exists(DHD_ACCOUNT_KEY):
        r = subprocess.run(["openssl", "genrsa", "-out", DHD_ACCOUNT_KEY, "2048"], capture_output=True, text=True, timeout=60)
        if r.returncode != 0:
            raise RuntimeError("ACME step 'account key' failed: openssl could not generate the account key")
    jwk = _rsa_jwk_from_private_key(DHD_ACCOUNT_KEY)
    thumb = _b64u(hashlib.sha256(json.dumps(
        {"e": jwk["e"], "kty": "RSA", "n": jwk["n"]}, separators=(",", ":"), sort_keys=True,
    ).encode()).digest())
    _s, _h, raw = _acme_http("GET", directory_url)
    try:
        directory = json.loads(raw.decode())
        new_nonce_url = directory["newNonce"]
        new_account_url = directory["newAccount"]
        new_order_url = directory["newOrder"]
    except Exception:
        raise RuntimeError("ACME step 'directory' failed: the ACME directory did not answer as expected at " + directory_url)
    state = {"nonce": ""}

    def fresh_nonce():
        _s, h, _b = _acme_http("HEAD", new_nonce_url)
        state["nonce"] = h.get("Replay-Nonce", "")

    def post(url, payload, kid=None, use_jwk=False):
        fresh_nonce()
        body = _acme_jws(DHD_ACCOUNT_KEY, url, payload, state["nonce"], kid=kid, jwk=(jwk if use_jwk else None))
        status, headers, raw = _acme_http("POST", url, body=body, ctype="application/jose+json")
        if headers.get("Replay-Nonce"):
            state["nonce"] = headers["Replay-Nonce"]
        data = None
        try:
            data = json.loads(raw.decode()) if raw else None
        except Exception:
            data = None
        return status, headers, data, raw

    status, headers, _d, _r = post(new_account_url, {"termsOfServiceAgreed": True}, use_jwk=True)
    account_url = headers.get("Location", "")
    if status not in (200, 201) or not account_url:
        raise RuntimeError("ACME step 'account' failed: the ACME server refused the account (HTTP " + str(status) + ")")
    status, order_headers, order, _r = post(new_order_url, {"identifiers": [{"type": "dns", "value": domain}]}, kid=account_url)
    if status not in (200, 201) or not isinstance(order, dict):
        raise RuntimeError("ACME step 'order' failed: the ACME server refused the order (HTTP " + str(status) + ")")
    order_url = order_headers.get("Location", "")
    authz_urls = order.get("authorizations") or []
    finalize_url = order.get("finalize", "")
    if not authz_urls or not finalize_url:
        raise RuntimeError("ACME step 'order' failed: the order carried no authorization")
    status, _h, authz, _r = post(authz_urls[0], None, kid=account_url)
    if not isinstance(authz, dict):
        raise RuntimeError("ACME step 'challenge' failed: the authorization did not load")
    challenge = None
    for c in authz.get("challenges") or []:
        if c.get("type") == "http-01":
            challenge = c
            break
    if not challenge:
        raise RuntimeError("ACME step 'challenge' failed: no HTTP-01 challenge was offered")
    token = str(challenge.get("token") or "")
    key_auth = token + "." + thumb
    with open(os.path.join(DHD_CHALLENGE_DIR, token), "w") as f:
        f.write(key_auth)
    status, _h, _d, _r = post(str(challenge.get("url") or ""), {}, kid=account_url)
    if status not in (200, 201):
        raise RuntimeError("ACME step 'challenge' failed: the ACME server refused the challenge response (HTTP " + str(status) + ")")
    valid = False
    for _ in range(30):
        time.sleep(1)
        _s, _h, a2, _r = post(authz_urls[0], None, kid=account_url)
        if isinstance(a2, dict) and a2.get("status") == "valid":
            valid = True
            break
        if isinstance(a2, dict) and a2.get("status") == "invalid":
            break
    if not valid:
        raise RuntimeError("ACME step 'challenge' failed: the domain proof did not validate — the usual cause is port 80 not reaching this machine (check the router forward) or the domain not resolving here yet")
    dhd_ensure_server_key()
    fd, csr_path = tempfile.mkstemp()
    os.close(fd)
    try:
        r = subprocess.run(
            ["openssl", "req", "-new", "-key", DHD_KEY_PATH, "-subj", "/CN=" + domain,
             "-addext", "subjectAltName=DNS:" + domain, "-outform", "DER", "-out", csr_path],
            capture_output=True, text=True, timeout=60,
        )
        if r.returncode != 0:
            raise RuntimeError("ACME step 'finalize' failed: openssl could not build the certificate request")
        with open(csr_path, "rb") as f:
            csr_b64 = _b64u(f.read())
    finally:
        try:
            os.unlink(csr_path)
        except Exception:
            pass
    status, _h, order2, _r = post(finalize_url, {"csr": csr_b64}, kid=account_url)
    if status not in (200, 201) or not isinstance(order2, dict):
        raise RuntimeError("ACME step 'finalize' failed: the ACME server refused the finalize (HTTP " + str(status) + ")")
    cert_url = order2.get("certificate", "")
    for _ in range(30):
        if cert_url:
            break
        if not order_url:
            break
        time.sleep(1)
        _s, _h, o3, _r = post(order_url, None, kid=account_url)
        if isinstance(o3, dict):
            cert_url = o3.get("certificate", "") or cert_url
            if o3.get("status") == "invalid":
                break
    if not cert_url:
        raise RuntimeError("ACME step 'download' failed: the order never produced a certificate")
    status, _h, _d, raw = post(cert_url, None, kid=account_url)
    if status != 200 or b"BEGIN CERTIFICATE" not in raw:
        raise RuntimeError("ACME step 'download' failed: the certificate did not download (HTTP " + str(status) + ")")
    os.makedirs(DHD_DIR, exist_ok=True)
    with open(DHD_CERT_PATH, "wb") as f:
        f.write(raw)
    dhd_write_meta("domain", domain)
    return True


# -- The proxy itself --------------------------------------------------

def dhd_gate_page(handler):
    # The DHD door's gate page, rendered through the ONE shared
    # renderer with the door's throttle state surfaced: attempts
    # remaining before lockout, and - while locked - the form
    # disabled with the remaining lockout time. The throttle itself
    # is unchanged; this only shows what it already knows. Build 2
    # adds the approval_mode fact: whether any trusted device
    # exists yet decides if a correct code here bootstraps or
    # knocks.
    ip = handler._client_ip()
    fails, lock_until = dhd_throttle_state(ip)
    host = handler.headers.get("Host") or ""
    approval = bool(door_approved_devices(load_config()))
    if lock_until > time.time():
        return gate_page_html(
            "Too many wrong codes - this door is locked.",
            door="dhd",
            name=host,
            lock_seconds=int(lock_until - time.time()),
            approval_mode=approval,
        )
    if fails > 0:
        return gate_page_html(
            "",
            door="dhd",
            name=host,
            attempts_remaining=max(0, DHD_LOCKOUT_FAILS - fails),
            approval_mode=approval,
        )
    return gate_page_html("", door="dhd", name=host, approval_mode=approval)


def dhd_throttle_state(ip):
    with DHD_LOCK:
        st = DHD_THROTTLE.get(ip)
        if not st:
            return 0, 0.0
        return st.get("fails", 0), st.get("lock_until", 0.0)


def dhd_throttle_fail(ip):
    with DHD_LOCK:
        st = DHD_THROTTLE.setdefault(ip, {"fails": 0, "lock_until": 0.0})
        st["fails"] = st.get("fails", 0) + 1
        fails = st["fails"]
        if fails >= DHD_LOCKOUT_FAILS:
            st["lock_until"] = time.time() + DHD_LOCKOUT_SECONDS
        return fails


def dhd_throttle_clear(ip):
    with DHD_LOCK:
        DHD_THROTTLE.pop(ip, None)


class DhdProxyHandler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"
    server_version = "ApexDHD"

    def log_message(self, fmt, *args):
        pass

    def _send_raw(self, code, body, ctype="text/html; charset=utf-8", extra=None):
        if isinstance(body, str):
            body = body.encode()
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        # Exactly ONE no-store header (build .91): a forwarded
        # response arrives with the bridge's own Cache-Control in
        # extra, and sending this one too put the pair on the wire.
        _extra_has_cc = bool(extra) and any(str(k).lower() == "cache-control" for k in extra)
        if not _extra_has_cc:
            self.send_header("Cache-Control", "no-store")
        if extra:
            for k, v in extra.items():
                self.send_header(k, v)
        self.end_headers()
        if self.command != "HEAD":
            try:
                self.wfile.write(body)
            except Exception:
                pass

    def _client_ip(self):
        try:
            return str(self.client_address[0] or "")
        except Exception:
            return ""

    def _accepted(self):
        # Build 2 (#173): acceptance is the device registry's word
        # — a trusted device's own token, or (only while no device
        # has ever been approved) the shared-code cookie. An
        # accepted device's last-seen and sliding renewal are
        # touched here, throttled inside door_touch_device.
        cfg = load_config()
        if not public_settings(cfg)["access_code"]:
            return False
        ok, dev, _legacy = door_gate_acceptance(cfg, door_cookie_value(self.headers.get("Cookie")))
        if ok and dev is not None:
            door_touch_device(cfg, dev)
        if ok:
            # Build 3 (#176): a DHD session starting is a pull
            # trigger — the phone's arrival refreshes the room.
            # The call is debounced inside; it fires at most one
            # pull per session gap and never blocks the door.
            sync_note_dhd_session((dev or {}).get("id") or "legacy")
        return ok

    def _pending_poll(self):
        # A waiting device's poll, answered by the proxy itself
        # (it lives in this same process): the pending id is the
        # only credential, and approval is the moment the device's
        # own cookie is set.
        q = urllib.parse.parse_qs(urllib.parse.urlsplit(self.path).query)
        pid = (q.get("id") or [""])[0]
        status, dev = door_pending_poll(load_config(), pid)
        extra = None
        if status == "approved" and dev is not None and dev.get("token"):
            extra = {"Set-Cookie": door_cookie_header(dev["token"])}
        self._send_raw(200, json.dumps({"status": status}), "application/json", extra=extra)

    def _serve_challenge(self, path):
        token = path.rsplit("/", 1)[-1]
        if not token or "/" in token or ".." in token:
            self._send_raw(404, "not found", "text/plain")
            return
        fp = os.path.join(DHD_CHALLENGE_DIR, token)
        try:
            with open(fp) as f:
                body = f.read()
            self._send_raw(200, body, "text/plain")
        except Exception:
            self._send_raw(404, "not found", "text/plain")

    def _forward(self):
        length = int(self.headers.get("Content-Length") or 0)
        body = self.rfile.read(length) if length > 0 else None
        headers = {}
        for k, v in self.headers.items():
            lk = k.lower()
            if lk in ("host", "connection", "content-length", "keep-alive", "proxy-connection", "te", "trailer", "transfer-encoding", "upgrade"):
                continue
            headers[k] = v
        headers["Host"] = "127.0.0.1:" + str(BRIDGE_PORT)
        headers["X-Forwarded-For"] = self._client_ip()
        headers["X-Forwarded-Proto"] = "https"
        headers["X-Apex-Via"] = "dhd"
        conn = http.client.HTTPConnection("127.0.0.1", BRIDGE_PORT, timeout=60)
        try:
            conn.request(self.command, self.path, body=body, headers=headers)
            resp = conn.getresponse()
            data = resp.read()
            out_headers = {}
            for k, v in resp.getheaders():
                lk = k.lower()
                if lk in ("connection", "content-length", "keep-alive", "transfer-encoding"):
                    continue
                out_headers[k] = v
            self._send_raw(resp.status, data, resp.getheader("Content-Type") or "application/octet-stream", extra=out_headers)
        except Exception:
            self._send_raw(502, "Apex is not answering behind the DHD door right now.", "text/plain")
        finally:
            try:
                conn.close()
            except Exception:
                pass

    def _gate_post(self):
        ip = self._client_ip()
        _fails, lock_until = dhd_throttle_state(ip)
        approval = bool(door_approved_devices(load_config()))
        if lock_until > time.time():
            wait = int(lock_until - time.time())
            self._send_raw(429, gate_page_html("Too many wrong codes - this door is locked.", door="dhd", name=self.headers.get("Host") or "", lock_seconds=wait, approval_mode=approval))
            return
        length = int(self.headers.get("Content-Length") or 0)
        raw = self.rfile.read(length) if length > 0 else b""
        try:
            form = urllib.parse.parse_qs(raw.decode("utf-8", "replace"))
            given = (form.get("code") or [""])[0].strip()
            dev_name = (form.get("device_name") or [""])[0].strip()
        except Exception:
            given = ""
            dev_name = ""
        cfg = load_config()
        want = public_settings(cfg)["access_code"]
        if want and given.upper() == want.upper():
            dhd_throttle_clear(ip)
            # Build 2 (#173): a correct code knocks. The outcome
            # is the registry's: a session with the device's OWN
            # token (the bootstrap, or a renewal), or a pending
            # device and its waiting room.
            action, dev = door_gate_outcome(cfg, dev_name, self.headers.get("User-Agent"), door_cookie_value(self.headers.get("Cookie")))
            if action == "session":
                self._send_raw(302, "", "text/plain", extra={"Location": "/", "Set-Cookie": door_cookie_header(dev["token"])})
            else:
                self._send_raw(200, gate_pending_html(dev["id"], dev["name"], door="dhd"))
            return
        fails = dhd_throttle_fail(ip)
        if fails >= DHD_LOCKOUT_FAILS:
            self._send_raw(429, gate_page_html("Too many wrong codes - this door is now locked.", door="dhd", name=self.headers.get("Host") or "", lock_seconds=DHD_LOCKOUT_SECONDS, approval_mode=approval))
            return
        # Progressive delay: each wrong code makes the next answer
        # slower (1s, 2s, 4s, capped at 8s).
        time.sleep(min(2 ** (fails - 1), 8))
        self._send_raw(200, gate_page_html("That code did not open the door.", door="dhd", name=self.headers.get("Host") or "", attempts_remaining=max(0, DHD_LOCKOUT_FAILS - fails), approval_mode=approval))

    def _handle(self):
        path = self.path.split("?")[0]
        is_tls = isinstance(self.connection, ssl.SSLSocket)
        if not is_tls:
            # Port 80: the ACME challenge path, and nothing else
            # but a redirect behind it.
            if path.startswith("/.well-known/acme-challenge/"):
                self._serve_challenge(path)
                return
            host = self.headers.get("Host") or ""
            target = "https://" + host + self.path if host else "/"
            self._send_raw(301, "", "text/plain", extra={"Location": target})
            return
        if path.startswith("/.well-known/acme-challenge/"):
            self._serve_challenge(path)
            return
        if path == "/__gate/pending" and self.command == "GET":
            # A waiting device's poll: answered without acceptance
            # (the pending id is its only credential) and never
            # forwarded — the registry lives in this process.
            self._pending_poll()
            return
        if path == "/__gate":
            if self.command == "POST":
                self._gate_post()
            else:
                self._send_raw(200, dhd_gate_page(self))
            return
        if path.startswith("/mesh/"):
            # Build .86: Mesh NEVER rides the DHD door. Mesh has
            # its own dedicated TLS door (its listen ports, its
            # own certificate) in every configuration, so the
            # proxy refuses /mesh/* outright — it never serves it
            # and never forwards it to the dashboard.
            self._send_raw(403, json.dumps({"ok": False, "reason": "unauthorized"}), "application/json")
            return
        if not self._accepted():
            if path.startswith("/api/"):
                self._send_raw(403, json.dumps({"error": "access code required"}), "application/json")
            else:
                self._send_raw(200, dhd_gate_page(self))
            return
        self._forward()

    do_GET = _handle
    do_POST = _handle
    do_PUT = _handle
    do_DELETE = _handle
    do_PATCH = _handle
    do_HEAD = _handle


def dhd_start_proxy():
    # Brings both listeners up with the certificate on disk.
    # Raises RuntimeError naming the step on failure.
    global DHD_SERVERS
    dhd_stop_proxy()
    if not os.path.exists(DHD_CERT_PATH):
        raise RuntimeError("DHD could not start at the TLS step: no certificate is on disk")
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    try:
        ctx.load_cert_chain(DHD_CERT_PATH, DHD_KEY_PATH)
    except Exception as e:
        raise RuntimeError("DHD could not start at the TLS step: the certificate could not be loaded (" + str(e) + ")")
    try:
        http_server = ThreadingHTTPServer(("0.0.0.0", 80), DhdProxyHandler)
    except PermissionError:
        raise RuntimeError("DHD could not start at the proxy step: ports below 1024 are reserved on Linux - grant the bridge the low-port capability once (sudo setcap 'cap_net_bind_service=+ep' on the python3 binary), then flip the switch again")
    except OSError as e:
        raise RuntimeError("DHD could not start at the proxy step: port 80 is not free (" + str(e) + ") - something else on this machine is using it")
    try:
        https_server = ThreadingHTTPServer(("0.0.0.0", 443), DhdProxyHandler)
        https_server.socket = ctx.wrap_socket(https_server.socket, server_side=True)
    except PermissionError:
        try:
            http_server.server_close()
        except Exception:
            pass
        raise RuntimeError("DHD could not start at the proxy step: ports below 1024 are reserved on Linux - grant the bridge the low-port capability once (sudo setcap 'cap_net_bind_service=+ep' on the python3 binary), then flip the switch again")
    except OSError as e:
        try:
            http_server.server_close()
        except Exception:
            pass
        raise RuntimeError("DHD could not start at the proxy step: port 443 is not free (" + str(e) + ") - something else on this machine is using it")
    DHD_SERVERS = [http_server, https_server]
    for srv in DHD_SERVERS:
        threading.Thread(target=srv.serve_forever, daemon=True).start()
    DHD_STATE["running"] = True
    return True


def dhd_stop_proxy():
    global DHD_SERVERS
    for srv in DHD_SERVERS:
        try:
            srv.shutdown()
        except Exception:
            pass
        try:
            srv.server_close()
        except Exception:
            pass
    DHD_SERVERS = []
    DHD_STATE["running"] = False


def dhd_self_check():
    # After the proxy starts: it must answer on 443 from this
    # machine. The gate page answering IS the healthy answer for
    # an unauthenticated local probe.
    try:
        ctx = ssl._create_unverified_context()
        conn = http.client.HTTPSConnection("127.0.0.1", 443, timeout=10, context=ctx)
        conn.request("GET", "/__gate")
        resp = conn.getresponse()
        resp.read()
        conn.close()
        return resp.status == 200
    except Exception:
        return False


def dhd_domain_resolves_here(domain):
    try:
        infos = socket.getaddrinfo(domain, 443, proto=socket.IPPROTO_TCP)
        ips = set()
        for info in infos:
            ips.add(str(info[4][0]))
        wan_ip, _wan_state = wan_ipv4()
        return lan_ipv4() in ips or "127.0.0.1" in ips or (bool(wan_ip) and wan_ip in ips)
    except Exception:
        return False


def dhd_bring_up(cfg, reason="enable", progress=None):
    # The one-click ladder, in order: certificate, proxy, check.
    # A failed step records its name and leaves the previous live
    # state untouched - a proxy already running keeps running. The
    # optional progress callback is the operation-card feed: the
    # same ladder, reported step by step, never a second path.
    ds = dhd_settings(cfg)
    domain = ds["domain"]
    stage = "certificate"
    DHD_STATE["starting"] = True

    def mark(step, state, detail="", commands=None):
        if progress:
            progress(step, state, detail, commands or [])

    try:
        mark("certificate", "running", "Preparing the certificate.")
        meta = dhd_cert_meta()
        _exp, days_left = dhd_cert_expiry()
        have_domain_cert = meta.get("mode") == "domain" and meta.get("domain") == domain and os.path.exists(DHD_CERT_PATH)
        if domain and not have_domain_cert:
            try:
                acme_obtain(domain, ds["acme_directory"])
            except Exception as e:
                # The door still opens today on a self-signed
                # certificate; the failure is named, not hidden.
                DHD_STATE["last_error"] = str(e)
                if not os.path.exists(DHD_CERT_PATH) or meta.get("mode") == "domain":
                    dhd_generate_self_signed(domain)
        elif domain and have_domain_cert and days_left is not None and days_left <= 30:
            try:
                acme_obtain(domain, ds["acme_directory"])
            except Exception as e:
                DHD_STATE["last_error"] = "Renewal attempt failed (the current certificate still has " + str(days_left) + " days): " + str(e)
        elif not domain:
            if not os.path.exists(DHD_CERT_PATH) or meta.get("mode") != "self-signed":
                dhd_generate_self_signed("")
        mark("certificate", "done", DHD_STATE["last_error"] if domain and not have_domain_cert else "Certificate ready.")
        stage = "proxy"
        mark("proxy", "running", "Starting Apex's reverse proxy on ports 80 and 443.")
        was_running = DHD_STATE["running"]
        if was_running and reason == "enable":
            # Re-enable with Apex's own proxy already listening is a
            # healthy state, never a port problem (#59): the door is
            # already working, so there is nothing to restart.
            mark("proxy", "done", "Apex's DHD proxy is already listening on 80/443 - this door working.")
        else:
            try:
                dhd_start_proxy()
            except Exception:
                if was_running:
                    # Reload failed; try to restore the previous face.
                    try:
                        dhd_start_proxy()
                    except Exception:
                        pass
                raise
            mark("proxy", "done", "The proxy is listening and forwarding to the bridge on loopback.")
        stage = "reachability"
        mark("reachability", "running", "Local check: confirming the proxy answers on 443 from this machine (answered on this machine proves the local half only - outside reach is reported separately).")
        meta = dhd_cert_meta()
        DHD_STATE["mode"] = meta.get("mode", "self-signed")
        if not dhd_self_check():
            raise RuntimeError("DHD started, but the reachability self-check failed: the proxy did not answer on 443 from this machine")
        if domain and DHD_STATE["mode"] != "domain":
            # Certificate pending is a degraded state, named here
            # so the page and the Servers dot can show it.
            if not DHD_STATE["last_error"]:
                DHD_STATE["last_error"] = "Domain saved - its certificate is still pending."
        elif DHD_STATE["mode"] == "domain":
            DHD_STATE["last_error"] = ""
        return True
    except Exception as e:
        DHD_STATE["last_error"] = str(e)
        mark(stage, "failed", str(e), dhd_failure_commands(stage, str(e), domain))
        return False
    finally:
        DHD_STATE["starting"] = False


def _dhd_on_summary(payload):
    # One honest sentence about a live DHD, shared by the operation
    # cards, the chat replies, and the bridge's own startup line.
    mode = payload.get("mode", "self-signed")
    if payload.get("publicVerified"):
        return "DHD is on at " + payload["publicUrl"] + " (" + mode + ") - outside reach verified."
    if payload.get("publicUrl"):
        return "DHD is on (" + mode + "). On this network: " + (payload.get("lanUrl") or "this machine") + ". Public address " + payload["publicUrl"] + " - outside reach unverified: " + (payload.get("nextStep") or "")
    return "DHD is on (" + mode + "). On this network: " + (payload.get("lanUrl") or "this machine") + ". No public address yet - outside reach unverified: " + (payload.get("nextStep") or "")


def dhd_state_payload(cfg, for_loopback):
    ds = dhd_settings(cfg)
    meta = dhd_cert_meta()
    expires, days_left = dhd_cert_expiry()
    active = bool(DHD_STATE["running"])
    mode = "off" if not ds["enabled"] else (meta.get("mode", "self-signed") if active else "off")
    cert_pending = bool(ds["domain"]) and mode != "domain"
    if not ds["enabled"]:
        status = "off"
    elif DHD_STATE["starting"]:
        status = "starting"
    elif not active:
        status = "degraded"
    elif cert_pending:
        status = "degraded"
    else:
        status = "healthy"
    # The honest address model (build .76, #59). Three different
    # addresses, never conflated: the LAN address (this network
    # only), the detected WAN address (this home's public IP, seen
    # by an outside echo - shown, but outside reach stays
    # UNVERIFIED until it is proven), and the domain (the public
    # address once its certificate is live AND the name resolves
    # to this machine - only then is outside reach verified).
    lan_ip = lan_ipv4()
    lan_url = ("https://" + lan_ip) if (ds["enabled"] and active) else ""
    wan_ip, wan_state = "", "undetected"
    if ds["enabled"] and active:
        wan_ip, wan_state = wan_ipv4()
    domain_live = bool(ds["domain"]) and mode == "domain"
    resolves_here = dhd_domain_resolves_here(ds["domain"]) if domain_live else False
    public_verified = bool(domain_live and resolves_here)
    public_url = ""
    if ds["enabled"] and active:
        if public_verified:
            public_url = "https://" + ds["domain"]
        elif wan_state == "public" and wan_ip:
            public_url = "https://" + wan_ip
    if public_verified:
        outside_reach = "verified"
        next_step = ""
        wan_note = ""
    elif not (ds["enabled"] and active):
        outside_reach = "none"
        next_step = ""
        wan_note = ""
    else:
        outside_reach = "unverified"
        if wan_state == "behind-nat":
            wan_note = "The address the internet sees for this home (" + wan_ip + ") is itself a private or carrier-NAT address - this machine sits behind another router/NAT, so it is not directly reachable from outside."
            next_step = "Forward ports 443 and 80 on the outer router to this machine, or add your domain - outside reach stays unverified until the public path is proven."
        elif wan_state == "undetected":
            wan_note = "Apex could not detect this home's public address (the echo lookup did not answer)."
            next_step = "Forward ports 443 and 80 on your router to this machine, or add your domain - outside reach stays unverified until the public path is proven."
        else:
            wan_note = ""
            next_step = "Forward ports 443 and 80 on your router to this machine (" + lan_ip + "), or add your domain - outside reach stays unverified until the public path is proven."
    return {
        "enabled": ds["enabled"],
        "active": active,
        "domain": ds["domain"],
        "mode": mode,
        "status": status,
        "publicUrl": public_url,
        "lanIp": lan_ip,
        "lanUrl": lan_url,
        "wanIp": wan_ip,
        "wanState": wan_state,
        "wanNote": wan_note,
        "publicVerified": public_verified,
        "outsideReach": outside_reach,
        "nextStep": next_step,
        "domainResolvesHere": resolves_here,
        "certExpiresAt": expires,
        "certDaysLeft": days_left,
        "certPending": cert_pending and ds["enabled"],
        "lastError": DHD_STATE["last_error"],
        "accessCode": public_settings(cfg)["access_code"] if for_loopback else None,
        "prereqs": dhd_prereq_check(),
        "lastOperation": op_latest("dhd-enable"),
    }


def dhd_renewal_loop():
    # Renewal is automatic: checked when the bridge starts (this
    # thread's first pass) and daily after that. A domain cert at
    # 30 days remaining or less is renewed with no operator
    # action; a self-signed cert is regenerated the same way.
    while True:
        try:
            cfg = load_config()
            ds = dhd_settings(cfg)
            if ds["enabled"] and DHD_STATE["running"]:
                _exp, days_left = dhd_cert_expiry()
                if days_left is not None and days_left <= 30:
                    dhd_bring_up(cfg, reason="renewal")
        except Exception:
            pass
        time.sleep(86400)


# -- DHD prerequisites, remedies, and operation cards (build .74) --
# The installer lands DHD's prerequisites up front; this section is
# the safety net and the shared operation engine. A check names
# each missing piece, an install action uses the same distro-family
# package step as the installer, and every failure that has a
# manual remedy carries the exact copy-ready commands for this
# machine's family. Operations are deterministic bridge state: no
# model call is made while one runs, steps and outputs stay inline,
# and a finished operation is persisted so its card remains in the
# chat thread after the work ends. APEX_DHD_MISSING and
# APEX_DHD_INSTALL_FAIL are diagnostics/test overrides only; unset
# in normal operation.

OPS_PATH = os.path.join(ROOT, "config", "operations.json")
OPS = {}
OPS_LOADED = False
OPS_LOCK = threading.Lock()
OP_EVENTS = {}
OP_RESUME = {}


def _dhd_env_set(name):
    out = set()
    for part in str(os.environ.get(name, "")).split(","):
        part = part.strip().lower().replace("_", "-")
        if part:
            out.add(part)
    return out


DHD_SIM_MISSING = _dhd_env_set("APEX_DHD_MISSING")
DHD_SIM_INSTALL_FAIL = _dhd_env_set("APEX_DHD_INSTALL_FAIL")


def detect_distro_family():
    data = {}
    try:
        with open("/etc/os-release") as f:
            for line in f:
                if "=" in line:
                    k, v = line.strip().split("=", 1)
                    data[k] = v.strip().strip('"')
    except Exception:
        pass
    distro_id = str(data.get("ID", "")).lower()
    like = str(data.get("ID_LIKE", "")).lower()
    label = str(data.get("PRETTY_NAME") or distro_id or "Unrecognized Linux")
    if distro_id == "bazzite":
        return "bazzite", label
    if distro_id in ("debian", "ubuntu", "pop", "linuxmint", "raspbian") or "debian" in like or "ubuntu" in like:
        return "debian", label
    if distro_id in ("fedora", "rhel", "centos") or "fedora" in like:
        return "fedora", label
    if distro_id in ("arch", "manjaro", "endeavouros") or "arch" in like:
        return "arch", label
    return "unknown", label


def _python3_path():
    p = shutil.which("python3") or sys.executable or "python3"
    try:
        return os.path.realpath(p)
    except Exception:
        return p


def dhd_lowport_commands():
    return ["sudo setcap 'cap_net_bind_service=+ep' " + chr(34) + _python3_path() + chr(34)]


def dhd_manual_commands(piece, family=None):
    if family is None:
        family, _label = detect_distro_family()
    if piece == "openssl":
        if family == "debian":
            return ["sudo apt-get update", "sudo apt-get install -y openssl"]
        if family == "fedora":
            return ["sudo dnf install -y openssl"]
        if family == "arch":
            return ["sudo pacman -S --needed openssl"]
        if family == "bazzite":
            return [
                "# Bazzite does not layer host packages; use its Fedora distrobox for the remedy:",
                "distrobox enter fedora -- sudo dnf install -y openssl",
            ]
        return [
            "# Install the openssl package with this distro's package manager, for example:",
            "sudo apt-get install -y openssl",
        ]
    if piece == "proxy":
        return [
            "# The DHD proxy is built into bridge.py; there is no separate proxy package to install.",
            "# Reinstall or update Apex so bridge.py is whole, then flip the DHD switch again.",
        ]
    if piece == "certhome":
        return [
            'sudo install -d -o "$USER" -g "$(id -gn)" "' + DHD_DIR + '"',
            'chmod -R u+rwX "' + DHD_DIR + '"',
        ]
    if piece == "lowport":
        return dhd_lowport_commands()
    if piece in ("port80", "port443"):
        port = "80" if piece == "port80" else "443"
        return [
            "sudo ss -ltnp 'sport = :" + port + "'",
            "# Stop the process named above (or move it to another port), then flip the DHD switch again.",
        ]
    return []


def dhd_failure_commands(stage, error, domain=""):
    text = str(error or "")
    low = text.lower()
    family, _label = detect_distro_family()
    if "cap_net_bind_service" in low or "ports below 1024" in low or "permission" in low and ("proxy step" in low or "port" in low):
        return dhd_lowport_commands()
    if "port 80 is not free" in low:
        return dhd_manual_commands("port80", family)
    if "port 443 is not free" in low:
        return dhd_manual_commands("port443", family)
    if "openssl" in low or "certificate" in low or "acme step" in low:
        return dhd_manual_commands("openssl", family)
    if stage == "reachability" and domain:
        return [
            'dig +short "' + domain + '"',
            'curl -I "http://' + domain + '/.well-known/acme-challenge/test"',
            "# The first command should return this machine's public IP; the second should reach Apex on port 80.",
        ]
    if stage == "proxy":
        return dhd_manual_commands("proxy", family)
    return []


def _dhd_port_state(port):
    if DHD_STATE.get("running"):
        return "ok", "In use by Apex's DHD proxy - this door working."
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 0)
        sock.bind(("0.0.0.0", port))
        return "ok", "Port " + str(port) + " is free."
    except PermissionError:
        return "permission", "Port " + str(port) + " needs the low-port capability on this Linux setup."
    except OSError as e:
        return "busy", "Port " + str(port) + " is already in use (" + str(e) + ")."
    finally:
        try:
            sock.close()
        except Exception:
            pass


def dhd_prereq_check():
    family, label = detect_distro_family()
    out = []
    proxy_ok = "DhdProxyHandler" in globals() and hasattr(ssl, "SSLContext") and "proxy" not in DHD_SIM_MISSING
    out.append({
        "id": "proxy", "name": "Reverse proxy component", "ok": bool(proxy_ok),
        "detail": "Built into bridge.py and present." if proxy_ok else "The proxy component did not pass Apex's self-test.",
        "installable": True, "commands": dhd_manual_commands("proxy", family),
    })
    openssl_path = shutil.which("openssl")
    openssl_ok = bool(openssl_path) and "openssl" not in DHD_SIM_MISSING
    openssl_detail = "Not found on PATH."
    if openssl_path:
        try:
            v = subprocess.run([openssl_path, "version"], capture_output=True, text=True, timeout=10).stdout.strip()
            openssl_detail = (v or openssl_path) + " is present."
        except Exception:
            openssl_detail = openssl_path + " is present."
    if not openssl_ok and openssl_path:
        openssl_detail = "Apex's prerequisite check is holding openssl as missing (diagnostics override)."
    out.append({
        "id": "openssl", "name": "OpenSSL binary", "ok": bool(openssl_ok),
        "detail": openssl_detail,
        "installable": family in ("debian", "fedora", "arch"),
        "commands": dhd_manual_commands("openssl", family),
    })
    cert_ok = "certhome" not in DHD_SIM_MISSING and "cert-home" not in DHD_SIM_MISSING
    cert_detail = "Certificate home is writable: " + DHD_DIR
    if cert_ok:
        try:
            os.makedirs(DHD_DIR, exist_ok=True)
            probe = os.path.join(DHD_DIR, ".apex-write-test-" + str(os.getpid()) + "-" + str(threading.get_ident()) + "-" + str(time.time_ns()))
            with open(probe, "w") as f:
                f.write("ok")
            os.unlink(probe)
        except Exception as e:
            cert_ok = False
            cert_detail = "Certificate home is not writable: " + str(e)
    else:
        cert_detail = "Apex's prerequisite check is holding the certificate home as unavailable (diagnostics override)."
    out.append({
        "id": "certhome", "name": "Certificate home", "ok": bool(cert_ok),
        "detail": cert_detail, "installable": True,
        "commands": dhd_manual_commands("certhome", family),
    })
    for port, piece in ((80, "port80"), (443, "port443")):
        state, detail = _dhd_port_state(port)
        commands = dhd_lowport_commands() if state == "permission" else dhd_manual_commands(piece, family)
        out.append({
            "id": piece, "name": "Port " + str(port) + " availability", "ok": state == "ok",
            "detail": detail, "installable": False, "commands": commands,
        })
    return {"family": family, "label": label, "items": out}


def _dhd_install_argv(piece, family):
    if piece == "openssl":
        if family == "debian":
            return ["apt-get update", "apt-get install -y openssl"]
        if family == "fedora":
            return ["dnf install -y openssl"]
        if family == "arch":
            return ["pacman -S --needed --noconfirm openssl"]
    return []


def _dhd_run_install_command(command, sudo_password=""):
    if os.geteuid() == 0:
        r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=300, cwd=shell_cwd())
        return r.returncode, (r.stdout + r.stderr).strip()
    result = run_shell("sudo " + command, timeout=300, sudo=True, sudo_password=sudo_password)
    return int(result.get("exit_code", 1)), str(result.get("output", "")).strip()


def dhd_install_prereqs(pieces=None, sudo_password=""):
    check = dhd_prereq_check()
    by_id = {item["id"]: item for item in check["items"]}
    wanted = pieces or [item["id"] for item in check["items"] if not item["ok"] and item["installable"]]
    results = []
    for piece in wanted:
        item = by_id.get(piece)
        if item is None:
            continue
        if item["ok"]:
            results.append({"id": piece, "ok": True, "detail": item["name"] + " is already present.", "output": "", "commands": item["commands"]})
            continue
        if piece in DHD_SIM_INSTALL_FAIL:
            results.append({"id": piece, "ok": False, "detail": "The install step for " + item["name"] + " failed (diagnostics override). Nothing else was changed.", "output": "Apex forced this install to fail so the manual remedy can be verified.", "commands": item["commands"]})
            continue
        output = ""
        ok = False
        try:
            if piece == "proxy":
                ok = "DhdProxyHandler" in globals() and hasattr(ssl, "SSLContext")
                if ok:
                    DHD_SIM_MISSING.discard("proxy")
                output = "Proxy component self-test passed." if ok else "Proxy component self-test still fails."
            elif piece == "openssl":
                commands = _dhd_install_argv(piece, check["family"])
                if not commands:
                    output = "This distro family has no automatic package step in Apex."
                else:
                    codes = []
                    chunks = []
                    for command in commands:
                        code, text = _dhd_run_install_command(command, sudo_password)
                        codes.append(code)
                        if text:
                            chunks.append(text[-1200:])
                    output = "\n".join(chunks)
                    ok = all(code == 0 for code in codes) and bool(shutil.which("openssl"))
                    if ok:
                        DHD_SIM_MISSING.discard("openssl")
            elif piece == "certhome":
                try:
                    os.makedirs(DHD_DIR, exist_ok=True)
                    probe = os.path.join(DHD_DIR, ".apex-write-test-" + str(os.getpid()) + "-" + str(threading.get_ident()) + "-" + str(time.time_ns()))
                    with open(probe, "w") as f:
                        f.write("ok")
                    os.unlink(probe)
                    ok = True
                    output = "Certificate home created and verified writable."
                except Exception:
                    if os.geteuid() != 0 or sudo_password:
                        cmd = "install -d -o " + str(os.getuid()) + " -g " + str(os.getgid()) + ' "' + DHD_DIR + '"' 
                        code, text = _dhd_run_install_command(cmd, sudo_password)
                        output = text
                        if code == 0:
                            DHD_SIM_MISSING.discard("certhome")
                            DHD_SIM_MISSING.discard("cert-home")
                            return dhd_install_prereqs([piece], sudo_password)
                    else:
                        output = "The bridge could not create or write the certificate home."
                if ok:
                    DHD_SIM_MISSING.discard("certhome")
                    DHD_SIM_MISSING.discard("cert-home")
            else:
                output = "This prerequisite has no automatic install step."
        except Exception as e:
            output = str(e)
            ok = False
        results.append({
            "id": piece, "ok": bool(ok),
            "detail": (item["name"] + " installed and verified.") if ok else ("The install step for " + item["name"] + " failed. Nothing else was changed."),
            "output": output[-2400:], "commands": item["commands"],
        })
    return {"ok": all(r["ok"] for r in results) if results else True, "results": results, "prereqs": dhd_prereq_check()}


def _ops_load():
    global OPS_LOADED
    if OPS_LOADED:
        return
    OPS_LOADED = True
    try:
        with open(OPS_PATH) as f:
            data = json.load(f)
        rows = data.get("operations", []) if isinstance(data, dict) else []
        for op in rows:
            if isinstance(op, dict) and op.get("id"):
                OPS[op["id"]] = op
    except Exception:
        pass


def _ops_save():
    try:
        os.makedirs(os.path.dirname(OPS_PATH), exist_ok=True)
        rows = list(OPS.values())[-25:]
        tmp = OPS_PATH + ".tmp"
        with open(tmp, "w") as f:
            json.dump({"operations": rows}, f)
        os.replace(tmp, OPS_PATH)
    except Exception:
        pass


def op_snapshot(op_id):
    _ops_load()
    with OPS_LOCK:
        op = OPS.get(op_id)
        return json.loads(json.dumps(op)) if op else None


def op_latest(kind=None):
    _ops_load()
    with OPS_LOCK:
        rows = list(OPS.values())
        if kind:
            rows = [op for op in rows if op.get("kind") == kind]
        return json.loads(json.dumps(rows[-1])) if rows else None


def op_new(kind, title, step_defs, source=""):
    _ops_load()
    op_id = "op-" + str(int(time.time() * 1000)) + "-" + secrets.token_hex(2)
    op = {
        "id": op_id, "kind": kind, "title": title, "source": source,
        "state": "running", "detail": "", "createdAt": int(time.time() * 1000),
        "updatedAt": int(time.time() * 1000),
        "steps": [{"id": sid, "name": name, "state": "pending", "detail": "", "commands": [], "output": ""} for sid, name in step_defs],
    }
    with OPS_LOCK:
        OPS[op_id] = op
        _ops_save()
    return op_id


def op_set(op_id, step_id, state, detail=None, commands=None, output=None, op_state=None):
    _ops_load()
    with OPS_LOCK:
        op = OPS.get(op_id)
        if not op:
            return
        for step in op.get("steps", []):
            if step.get("id") == step_id:
                step["state"] = state
                if detail is not None:
                    step["detail"] = detail
                if commands is not None:
                    step["commands"] = commands
                if output is not None:
                    step["output"] = output[-2400:]
                break
        if op_state:
            op["state"] = op_state
        op["updatedAt"] = int(time.time() * 1000)
        _ops_save()


def op_finish(op_id, state, detail):
    _ops_load()
    with OPS_LOCK:
        op = OPS.get(op_id)
        if not op:
            return
        op["state"] = state
        op["detail"] = detail
        op["updatedAt"] = int(time.time() * 1000)
        _ops_save()


def _dhd_op_fail(op_id, step_id, detail, commands=None):
    op_set(op_id, step_id, "failed", detail, commands=commands or [])
    op_finish(op_id, "failed", detail)


def _dhd_release_waiting_ops(ok, detail, commands=None):
    _ops_load()
    with OPS_LOCK:
        candidates = [op for op in OPS.values() if op.get("kind") == "dhd-enable" and op.get("state") == "waiting"]
    for op in candidates:
        OP_RESUME[op["id"]] = {"ok": bool(ok), "detail": detail, "commands": commands or []}
        event = OP_EVENTS.get(op["id"])
        if event:
            event.set()


def _run_dhd_enable_op(op_id, sudo_password=""):
    try:
        cfg = load_config()
        if dhd_settings(cfg)["enabled"] and DHD_STATE.get("running"):
            for step_id in ("prerequisites", "certificate", "proxy", "reachability"):
                op_set(op_id, step_id, "done", "Already live.")
            op_finish(op_id, "done", "DHD is already on.")
            return
        op_set(op_id, "prerequisites", "running", "Checking the proxy component, OpenSSL, certificate home, and ports 80/443.")
        check = dhd_prereq_check()
        missing = [item for item in check["items"] if not item["ok"]]
        installable = [item for item in missing if item["installable"]]
        blockers = [item for item in missing if not item["installable"]]
        if installable:
            names = ", ".join(item["name"] for item in installable)
            event = threading.Event()
            OP_EVENTS[op_id] = event
            op_set(op_id, "prerequisites", "waiting", "Missing: " + names + ". Use Install on this card or the DHD page; the enable continues automatically afterward.", op_state="waiting")
            event.wait()
            resume = OP_RESUME.pop(op_id, {"ok": False, "detail": "The install step did not report a result.", "commands": []})
            OP_EVENTS.pop(op_id, None)
            if not resume.get("ok"):
                _dhd_op_fail(op_id, "prerequisites", resume.get("detail") or "A prerequisite install failed.", resume.get("commands") or [])
                return
            check = dhd_prereq_check()
            missing = [item for item in check["items"] if not item["ok"]]
            blockers = [item for item in missing if not item["installable"]]
            installable = [item for item in missing if item["installable"]]
            if installable:
                commands = []
                for item in installable:
                    commands.extend(item["commands"])
                _dhd_op_fail(op_id, "prerequisites", "A prerequisite is still missing after the install step: " + ", ".join(item["name"] for item in installable), commands)
                return
        if blockers:
            commands = []
            detail_bits = []
            for item in blockers:
                commands.extend(item["commands"])
                detail_bits.append(item["name"] + ": " + item["detail"])
            _dhd_op_fail(op_id, "prerequisites", " ".join(detail_bits), commands)
            return
        op_set(op_id, "prerequisites", "done", "Every prerequisite is present.")
        cfg = load_config()
        dhd_save(cfg, {"enabled": True})
        ensure_access_code(cfg)
        cfg = load_config()
        if ACTIVE_BIND_HOST == "0.0.0.0":
            op_set(op_id, "certificate", "waiting", "The bridge is restarting onto loopback-only before the public proxy opens.", op_state="waiting")
            op_finish(op_id, "waiting", "Bridge restart requested; this operation resumes when the new bridge answers.")
            schedule_bridge_restart()
            return

        def progress(step_id, state, detail, commands=None):
            op_set(op_id, step_id, state, detail, commands=commands or [])

        ok = dhd_bring_up(cfg, progress=progress)
        cfg = load_config()
        payload = dhd_state_payload(cfg, True)
        if ok:
            op_set(op_id, "reachability", "done", "Local check passed: the proxy answered on 443 on this machine (answered on this machine). Outside reach is reported separately on the DHD page." + (" The domain certificate is still pending; the door is answering on the self-signed certificate meanwhile." if payload.get("certPending") else ""))
            summary = _dhd_on_summary(payload)
            op_finish(op_id, "done", summary)
        else:
            detail = DHD_STATE.get("last_error") or "DHD did not come up."
            op_finish(op_id, "failed", detail)
    except Exception as e:
        op_finish(op_id, "failed", "DHD enable failed: " + str(e))


def dhd_start_enable_operation(source, sudo_password=""):
    existing = op_latest("dhd-enable")
    if existing and existing.get("state") in ("running", "waiting"):
        return existing["id"]
    op_id = op_new("dhd-enable", "Turn on DHD", [
        ("prerequisites", "Prerequisites"), ("certificate", "Certificate"),
        ("proxy", "Reverse proxy"), ("reachability", "Reachability check"),
    ], source=source)
    threading.Thread(target=_run_dhd_enable_op, args=(op_id, sudo_password), daemon=True).start()
    return op_id


def dhd_resume_enable_operation():
    op = op_latest("dhd-enable")
    if not op or op.get("state") not in ("waiting", "running"):
        return
    cfg = load_config()
    if not dhd_settings(cfg)["enabled"]:
        return
    if DHD_STATE.get("running"):
        payload = dhd_state_payload(cfg, True)
        op_set(op["id"], "prerequisites", "done", "Every prerequisite is present.")
        op_set(op["id"], "certificate", "done", "Certificate ready (" + payload.get("mode", "self-signed") + ").")
        op_set(op["id"], "proxy", "done", "The proxy is listening on 80/443.")
        op_set(op["id"], "reachability", "done", "Local check passed: the proxy answered on 443 on this machine (answered on this machine).")
        op_finish(op["id"], "done", _dhd_on_summary(payload))
    elif DHD_STATE.get("last_error"):
        detail = DHD_STATE["last_error"]
        stage = "proxy" if "proxy step" in detail else "reachability" if "self-check" in detail else "certificate"
        op_set(op["id"], stage, "failed", detail, commands=dhd_failure_commands(stage, detail, dhd_settings(cfg)["domain"]))
        op_finish(op["id"], "failed", detail)


def dhd_chat_intent(text):
    raw = str(text or "").strip()
    low = " " + re.sub(r"[^a-z0-9]+", " ", raw.lower()).strip() + " "
    if " dhd " not in low:
        return None
    if re.match(r"\s*(how|what|why|explain)\b", raw, re.I):
        return None
    if re.search(r"\b(turn off|switch off|disable|stop|shut off)\b", low):
        return "disable"
    if re.search(r"\b(turn on|switch on|enable|start|open|activate|set up|setup)\b", low):
        return "enable"
    return None


# --- Navi Mesh — Ring 1 (build .85, issue #78) -------------------
# Navi Mesh is pooled inference across the operator's own machines:
# a stronger machine answers for a weaker one. Routing is whole-
# request: the borrower assembles the prompt exactly as it would
# for its own model (persona, Vault, RAG, history) and a paired
# lender's model writes the whole reply. Alex's rulings from the
# design call govern the defaults: borrowing is AUTOMATIC for
# paired peers once Mesh is on, with an off switch (R1); nothing
# anywhere is tied to a specific operator, machine, or domain —
# every name and address comes from config and peer cards (R2);
# a lender that drops mid-answer raises a notice AND the request
# falls back to a fresh local reply, never spliced prose (R3);
# lending is automatic when idle with an off switch that aborts
# an in-flight borrowed generation immediately (R4); a busy
# refusal gets a short configurable grace before local fallback,
# and the lender is only ever claimed on the NEXT request (R5).
#
# Hard rules (spec sections 4.3 and 7): /mesh/* is never part of
# the passcode-cookie dashboard surface — the DHD proxy hands it
# to this section directly, before its cookie gate; node keys and
# dashboard codes share no credentials in either direction; and
# the lender persists NOTHING but ledger metadata — no prompt or
# reply text is ever written to disk on the lending side.

MESH_DIR = os.path.join(ROOT, "config", "mesh")
MESH_KEY_PATH = os.path.join(MESH_DIR, "node_key.pem")
MESH_PUB_PATH = os.path.join(MESH_DIR, "node_pub.pem")
MESH_CERT_PATH = os.path.join(MESH_DIR, "cert.pem")
MESH_CERT_KEY_PATH = os.path.join(MESH_DIR, "cert_key.pem")
MESH_LEDGER_PATH = os.path.join(MESH_DIR, "ledger.jsonl")
# Build 4 (#178): the managed backup route's dedicated SSH
# keypair — separate from the node identity keypair, held at
# 0600 under config/mesh, and never part of any backup payload
# (backups collect named files; key files are never collected —
# the same custody as the node keypair).
MESH_TUNNEL_KEY_PATH = os.path.join(MESH_DIR, "tunnel_key")
MESH_TUNNEL_PUB_PATH = MESH_TUNNEL_KEY_PATH + ".pub"
# Where restored pairing-card pictures land (Build 9, #89): the
# GitHub backup's Mesh pairing cards restore into this folder —
# the pictures stay importable files, and the peer pictures are
# also re-paired through the normal card check on import.
MESH_CARDS_DIR = os.path.join(MESH_DIR, "cards")

MESH_LOCK = threading.Lock()
MESH_STATE = {
    # Lender side: the in-flight borrowed generation, if any. The
    # connection object is kept so instant-off can abort it by
    # closing the socket (R4: OFF means OFF mid-generation too).
    "borrowed_conn": None,
    "borrowed_resp": None,
    "borrowed_peer": "",
    "borrowed_started": 0.0,
    "lend_stop_stamp": 0.0,
    # Federation (Build 8): the fedinet an in-flight federation
    # borrow belongs to ("" for a pairwise borrow — those are
    # never preempted), and the owner-preemption stamp, the
    # preempt path's mirror of lend_stop_stamp.
    "borrowed_fedinet": "",
    "preempt_stamp": 0.0,
    # Local generations in flight on this machine (either role) —
    # the busy/idle signals admission and status read.
    "local_gen": 0,
    "last_local_done": 0.0,
    "last_borrowed_done": 0.0,
    # Borrower side: an outbound borrowed generation in flight.
    "borrow_out": False,
    "borrow_peer": "",
    # Reachability probe cache: nodeId -> (stamp, status-or-None,
    # error-kind). Status probes are short and chat is latency-
    # sensitive, so a probe is reused for a few seconds.
    "probe_cache": {},
    # Borrower side (build .87): nodeId -> stamp of the last
    # in-flight DROP this machine suffered from that peer — the
    # status payload's "dropped" peer state and the Servers row's
    # warning read it. A later successful probe supersedes it,
    # and a borrow the peer actually serves clears it outright.
    "peer_dropped": {},
    # Background peer health (Build 7, #87): whether the health
    # thread has been started, and nodeId -> the last health
    # state word the bridge LOGGED for that peer ("reachable" or
    # "unreachable:<kind>"). Only transitions are logged, so the
    # word lives here to compare against — the probe cache above
    # stays the status payload's source of truth.
    "health_started": False,
    "peer_health": {},
}
MESH_SERVERS = []


def _mesh_int(value, fallback):
    try:
        return int(value)
    except Exception:
        return fallback


def _mesh_addr_parts(entry, default_port):
    # 0.2.0e Build 4 (#143/#154): one address entry, split into
    # the (host, port) a dial actually uses. An entry is either a
    # bare host — it inherits the record's port, which is what
    # every card and record before this build carries — or
    # host:port, carrying its own. Every peer dial (borrow,
    # probe, health sweep) parses through here, so a mixed list
    # of bare and ported entries works everywhere at once, and a
    # legacy record works untouched.
    host = str(entry or "").strip()
    host = host.split("://")[-1].split("/")[0].strip()
    try:
        dport = int(default_port)
    except Exception:
        dport = 8765
    if host.startswith("[") and "]" in host:
        inner, _, rest = host[1:].partition("]")
        if rest.startswith(":") and rest[1:].isdigit():
            return inner, int(rest[1:])
        return inner, dport
    if host.count(":") == 1:
        h, _, p = host.partition(":")
        if h and p.isdigit():
            return h, int(p)
    return host, dport


def _mesh_endpoint_clean(value):
    # An operator-declared public endpoint in canonical
    # host:port shape, or "" — for "absent" and for anything
    # that is not exactly a host and a port. The endpoint is
    # typed by the operator, never harvested (Build 4, #154):
    # a bare host with no port is not an endpoint, because the
    # port is the half a public door cannot inherit from a card
    # minted for the LAN.
    s = str(value or "").strip()
    if not s:
        return ""
    host, port = _mesh_addr_parts(s, 0)
    if not host or not port or port < 1 or port > 65535:
        return ""
    if any(c in host for c in " \t"):
        return ""
    return host + ":" + str(port)


def _mesh_endpoint_error(value):
    # The settings-save refusal for a public endpoint that was
    # typed but is not a host:port — a sentence, in the named-
    # failure style, or "" when the value is fine (or empty).
    s = str(value or "").strip()
    if not s or _mesh_endpoint_clean(s):
        return ""
    return ("The public endpoint must be a host and a port and nothing else — "
            "like mesh.example.org:8443. Fix it, or clear the field to declare no endpoint.")


# -- Junk peer records + in-place repair (0.2.0e Build 6, #143) -----
# The field failure that darkened the first road: a peer record
# whose address list was harvested from the wrong machine —
# loopback, a container bridge's gateway, ghosts — so every dial
# terminated somewhere that was never the peer. Build 4 stopped
# MINTING such lists; records already carrying them sat dark with
# a verdict and homework. Build 6 names the junk and heals it in
# place: an entry is junk when, as a way to reach ANOTHER machine,
# it is a dead end by construction — loopback in any spelling,
# the unspecified address, a link-local, or the default Docker
# bridge gateway (172.17.0.1, the address that cost the field a
# week). Private LAN addresses, public addresses, and hostnames
# are never junk: they are somebody's real road.

def _mesh_addr_is_junk(entry):
    host, _port = _mesh_addr_parts(entry, 8765)
    h = str(host or "").strip().lower()
    if not h:
        return True
    if h in ("localhost", "::1", "0.0.0.0", "::"):
        return True
    if h.startswith("127."):
        return True
    if h == "172.17.0.1":
        return True
    try:
        addr = ipaddress.ip_address(h)
        if addr.is_loopback or addr.is_link_local or addr.is_unspecified:
            return True
    except Exception:
        pass
    return False


def mesh_record_repair(peer):
    # The repair offer for one peer record, or None when the
    # record's addresses could reach the peer. A repair exists
    # when the record HAS addresses and EVERY one is junk: the
    # record points nowhere, and no verdict about the peer's
    # identity can be drawn from dialing it — the pair was never
    # actually asked. (A record with no addresses at all is a
    # different state — a fresh card is simply its whole cure,
    # and the import path is already its affordance.)
    addrs = list(peer.get("addresses") or [])
    if not addrs:
        return None
    junk = [a for a in addrs if _mesh_addr_is_junk(a)]
    if len(junk) != len(addrs):
        return None
    return {
        "kind": "junk-addresses",
        "junkCount": len(junk),
        "sentence": ("Every address on " + peer["name"] + "'s card is a dead end — "
                     "loopback or container-bridge entries that lead back to a local "
                     "interface, never to " + peer["name"] + ". The record points "
                     "nowhere, so this pair has never actually been asked. Heal it "
                     "here: import a fresh card minted on " + peer["name"] + " itself, "
                     "or clear the dead addresses now and import the fresh card when "
                     "you have it — the pairing, its pin, and its history are kept "
                     "either way."),
    }


def mesh_card_refresh_repair(cfg, peer):
    # (v0.3.0e — Build 6, #227) The subset wound, named: a
    # federation-marked peer whose stored addresses are a
    # PROPER SUBSET of the addresses its verified gateway card
    # carries. The knock dials the record, never the card, so
    # every road the record is missing is a road never tried —
    # the first external join burned a day on exactly this (a
    # hand-paired LAN-only record under a gateway whose card
    # also named its public road). The heal is the card refresh
    # the federation import performs (#227 part 2), offered in
    # place under the health check's heal doctrine. Returns the
    # repair offer, or None when the record carries every road
    # its card names (or no verified card is on file).
    fed_id = str(peer.get("viaFedinet") or "")
    if not fed_id:
        return None
    fed = next((f for f in federation_settings(cfg)
                if f.get("id") == fed_id
                and f.get("gatewayNodeId") == peer.get("nodeId")), None)
    payload = fed.get("gatewayCardPayload") if fed else None
    if not isinstance(payload, dict):
        return None
    card_peer, _err = _fed_verify_gateway_card(payload)
    if card_peer is None:
        return None
    have = {_mesh_addr_parts(a, peer.get("port")) for a in (peer.get("addresses") or [])}
    want = {_mesh_addr_parts(a, card_peer.get("port")) for a in (card_peer.get("addresses") or [])}
    if not have or not have < want:
        return None
    missing = len(want - have)
    return {
        "kind": "card-refresh",
        "missingCount": missing,
        "sentence": (peer["name"] + "'s record carries only some of the addresses "
                     "its gateway card carries — " + str(missing) + " road"
                     + ("s" if missing != 1 else "") + " the card names "
                     + ("are" if missing != 1 else "is") + " missing here, and the knock "
                     "dials this record, never the card, so those roads are never "
                     "tried. The gateway card on file is signed and verified; heal "
                     "it here: refresh the record's addresses, port, pin, and name "
                     "from the card. The pairing's own facts — relay ticks, the "
                     "tunnel route, its history — are kept exactly as they stand."),
    }


def mesh_repair_peer_record(cfg, node_id):
    # (ok, sentence): strip the junk entries from one peer's
    # stored record, in place. The record itself — name, pin,
    # tunnel route, history — is untouched; only the dead-end
    # addresses go. A record with real addresses among the junk
    # is refused: that is an edit, not a repair, and edits go
    # through a fresh card.
    ms = mesh_settings(cfg)
    target = None
    for p in ms["borrow"]["peers"]:
        if p["nodeId"] == node_id:
            target = p
            break
    if target is None:
        return False, "No paired machine with that identity is on this machine."
    rep = mesh_record_repair(target)
    if rep is None:
        # (v0.3.0e — Build 6, #227): the second heal — a
        # federation-marked record missing roads its verified
        # gateway card carries is refreshed from that card.
        crep = mesh_card_refresh_repair(cfg, target)
        if crep is not None:
            fed = next((f for f in federation_settings(cfg)
                        if f.get("id") == str(target.get("viaFedinet") or "")), None)
            payload = fed.get("gatewayCardPayload") if fed else None
            card_peer, _err = _fed_verify_gateway_card(payload or {})
            if card_peer is None:
                return False, ("The gateway card on file for " + target["name"]
                               + " no longer verifies — import a fresh gateway "
                               "card from the Federation page and the record "
                               "can be refreshed from it.")
            if _mesh_peer_card_refresh(cfg, node_id, card_peer):
                save_config(cfg)
                return True, ("Refreshed — " + target["name"] + "'s record now "
                              "carries every road its gateway card names: "
                              "addresses, port, pin, and name from the verified "
                              "card. The pairing's own facts — relay ticks, the "
                              "tunnel route, its history — stand untouched.")
            return False, "No paired machine with that identity is on this machine."
        return False, (target["name"] + "'s record holds at least one address that "
                       "could reach it — there is nothing to repair. A fresh card "
                       "is the cure for a record that needs changing.")
    block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
    block = dict(block)
    borrow = block.get("borrow") if isinstance(block.get("borrow"), dict) else {}
    borrow = dict(borrow)
    peers = []
    for raw in borrow.get("peers") or []:
        if isinstance(raw, dict) and str(raw.get("nodeId") or "") == node_id:
            raw = dict(raw)
            raw["addresses"] = [a for a in (raw.get("addresses") or [])
                                if not _mesh_addr_is_junk(a)]
        peers.append(raw)
    borrow["peers"] = peers
    block["borrow"] = borrow
    cfg["mesh"] = block
    save_config(cfg)
    return True, ("Cleared — the dead addresses are gone from " + target["name"]
                  + "'s record. The pairing and its history stand; import a fresh "
                  "card minted on " + target["name"] + " itself and the road has "
                  "somewhere to go again.")


def _mesh_tunnel_sanitize(raw):
    # Build 4 (#178): a peer's managed backup route, cut down to
    # exactly the fields the supervisor needs. Off by default;
    # the local port is bridge-allocated and persisted once
    # chosen, so it is stable across restarts.
    if not isinstance(raw, dict):
        raw = {}
    host = str(raw.get("sshHost") or "").strip()
    if any(c in host for c in " \t/"):
        host = ""
    user = str(raw.get("sshUser") or "").strip()
    if any(c in user for c in " \t@/"):
        user = ""
    sport = _mesh_int(raw.get("sshPort"), 227)
    if sport < 1 or sport > 65535:
        sport = 227
    lp = raw.get("localPort")
    try:
        lp = int(lp)
    except Exception:
        lp = None
    if lp is not None and not (1024 < lp <= 65535):
        lp = None
    return {"enabled": raw.get("enabled") is True,
            "sshHost": host[:255], "sshPort": sport,
            "sshUser": user[:64], "localPort": lp}


def _mesh_peer(p):
    # One peer record, normalized. Addresses keep the operator's
    # order — LAN first as listed on the card (spec section 3.1).
    if not isinstance(p, dict):
        return None
    addrs = p.get("addresses")
    if not isinstance(addrs, list):
        addrs = []
    addrs = [str(a).strip() for a in addrs if str(a).strip()]
    return {
        "name": str(p.get("name") or "Unnamed peer"),
        "addresses": addrs,
        "port": _mesh_int(p.get("port"), 8765),
        "nodeId": str(p.get("nodeId") or ""),
        "publicKeyPem": str(p.get("publicKeyPem") or ""),
        "certFingerprint": str(p.get("certFingerprint") or "").strip().lower(),
        "canBorrow": p.get("canBorrow") is not False,
        "capabilities": _mesh_caps_sanitize(p.get("capabilities")),
        # Build 4 (#154): the endpoint the peer's operator
        # declared for it on its card — the one address of a
        # peer that is shown, because it was typed for
        # publication. "" when the card declared none.
        "publicEndpoint": _mesh_endpoint_clean(p.get("publicEndpoint")),
        # Build 4 (#178): the managed backup route (borrower-side
        # config; local to this machine, never on the card) and
        # the peer's tunnel PUBLIC key (card-carried, public
        # data — the private half never leaves the peer).
        "tunnel": _mesh_tunnel_sanitize(p.get("tunnel")),
        "tunnelPublicKey": str(p.get("tunnelPublicKey") or "").strip(),
        # 0.2.0e Build 7 (#154): relay consent — three flat lists
        # of node ids, LOCAL to this machine, never on the card,
        # each ticked here by this machine's operator:
        #   relayCarryTo     — "this peer (as relay) may carry MY
        #                      envelopes to these destinations"
        #                      (the asker's tick, on the relay's
        #                      record)
        #   relayAcceptFrom  — "this peer (as relay) may carry
        #                      envelopes TO ME from these askers"
        #                      (the lender's tick, on the relay's
        #                      record)
        #   relayWillCarryTo — "I will carry THIS peer's (the
        #                      asker's) envelopes to these
        #                      destinations" (the relay's own
        #                      tick, on the asker's record)
        # A carriage happens only when all three machines' ticks
        # agree for the triple; any missing tick refuses the
        # carriage by name.
        "relayCarryTo": _mesh_id_list(p.get("relayCarryTo")),
        "relayAcceptFrom": _mesh_id_list(p.get("relayAcceptFrom")),
        "relayWillCarryTo": _mesh_id_list(p.get("relayWillCarryTo")),
        # 0.2.0e Build 8 (#154): federation provenance — the
        # fedinet whose roster made this record, when the
        # federation made it. "" for a hand-made pairing (the
        # operator's own ceremony, never the federation's to
        # create, edit, or remove). autoPaired marks records the
        # federation materialized from a signed roster, as
        # opposed to the one introduction record an import makes.
        "viaFedinet": str(p.get("viaFedinet") or "")[:64],
        "autoPaired": p.get("autoPaired") is True,
    }


def _mesh_id_list(raw):
    # A consent list: node ids only, stripped, deduped, capped.
    if not isinstance(raw, list):
        return []
    out = []
    for v in raw:
        s = str(v or "").strip()
        if s and s not in out:
            out.append(s)
        if len(out) >= 64:
            break
    return out


def _mesh_caps_sanitize(raw):
    # A peer's capability self-report, cut down to short strings
    # and numbers only (build .86): a card is operator-carried
    # data, so nothing long, nested, or structural survives into
    # the stored record or the roster. None when absent.
    if not isinstance(raw, dict):
        return None
    def s(v, n=80):
        t = str(v or "").strip()
        if len(t) <= n:
            return t
        # Cut at a word boundary: a capability line must never
        # slice a word in half on the roster.
        cut = t[:n].rsplit(" ", 1)[0]
        return (cut or t[:n]) + "…"
    def num(v):
        try:
            return float(v)
        except Exception:
            return 0
    def i(v):
        try:
            return int(v)
        except Exception:
            return 0
    caps = raw.get("caps") if isinstance(raw.get("caps"), dict) else {}
    def cap_i(v, fallback):
        # Build 7 (Fix F): the caps on a card are the peer's
        # CONFIGURED values. A card that does not state them (one
        # written before caps rode along, or a hand-trimmed one)
        # falls back to the shipped defaults — a stated value
        # under 1 is not a real configuration (the settings save
        # validates 1..100 / 1..10000), so it reads as unstated
        # rather than as "lends at most 0%".
        n = i(v)
        return n if n >= 1 else fallback
    return {
        "hardwareClass": s(raw.get("hardwareClass"), 60),
        "gpu": s(raw.get("gpu")),
        "cpu": s(raw.get("cpu")),
        "ramGB": num(raw.get("ramGB")),
        "model": s(raw.get("model"), 24) or "navi",
        "speedClass": s(raw.get("speedClass"), 40),
        # A card written before the posture rode along (Build 6
        # and earlier) was an idle-only lender by construction —
        # absent reads as True, the posture those builds kept.
        "idleOnly": raw.get("idleOnly") is not False,
        "caps": {
            "maxSharePercent": cap_i(caps.get("maxSharePercent"), 25),
            "maxGenerationsPerHour": cap_i(caps.get("maxGenerationsPerHour"), 20),
        },
    }


def mesh_capabilities(cfg):
    # This node's capability self-report (build .86): an honest
    # summary of THIS machine from its own data — the stored
    # hardware profile (detected GPU kind/name, CPU model, total
    # RAM, tier) and the lend settings as caps. The model is
    # "navi"; the speed class derives from the GPU situation:
    # discrete GPU -> "GPU-accelerated", integrated ->
    # "Integrated graphics", none -> "CPU".
    ms = mesh_settings(cfg)
    hp = cfg.get("hardware_profile") if isinstance(cfg, dict) else None
    det = hp.get("detected") if isinstance(hp, dict) else {}
    if not isinstance(det, dict):
        det = {}
    gpu_kind = str(det.get("gpuKind") or "none").strip().lower()
    gpu_name = str(det.get("gpuName") or "").strip()[:80]
    cpu = str(det.get("cpuModel") or "").strip()[:80]
    try:
        ram = float(det.get("totalRamGB") or 0)
    except Exception:
        ram = 0
    tier = ""
    if isinstance(hp, dict):
        tier = str(hp.get("tierLabel") or hp.get("tier") or "").strip()[:60]
    if gpu_kind == "discrete":
        speed = "GPU-accelerated"
        gpu = gpu_name or "Discrete GPU"
    elif gpu_kind == "integrated":
        speed = "Integrated graphics"
        gpu = gpu_name or "Integrated graphics"
    else:
        speed = "CPU"
        gpu = ""
    return {
        "hardwareClass": tier,
        "gpu": gpu,
        "cpu": cpu,
        "ramGB": ram,
        "model": "navi",
        "speedClass": speed,
        # Build 7 (Fix F2): the idle posture rides the same
        # self-report as the caps, so a peer's card and roster
        # line state it truthfully — "lends when idle" vs "lends
        # even while busy" — from this machine's own setting.
        "idleOnly": ms["lend"]["idleOnly"],
        "caps": {
            "maxSharePercent": ms["lend"]["maxSharePercent"],
            "maxGenerationsPerHour": ms["lend"]["maxGenerationsPerHour"],
        },
    }


def guardian_settings(cfg):
    # Guardian (Build 14, #156): absent-block-safe, like
    # mesh_settings / dhd_settings — an install whose config
    # predates the guardian block reads as the shipped defaults:
    # ON, a 1024 MB reserve, a 2-second sample, kills allowed for
    # operator-named expendable processes (none named by default,
    # so nothing can be killed until the operator says what may).
    g = cfg.get("guardian") if isinstance(cfg, dict) else None
    if not isinstance(g, dict):
        g = {}
    expendable = g.get("expendable")
    if not isinstance(expendable, list):
        expendable = []
    expendable = [str(x).strip() for x in expendable if str(x).strip()]
    try:
        sample_seconds = float(g.get("sampleSeconds"))
    except Exception:
        sample_seconds = 2.0
    return {
        "enabled": g.get("enabled") is not False,
        "reserveMb": _mesh_int(g.get("reserveMb"), 1024),
        "sampleSeconds": sample_seconds,
        "slowGenSeconds": _mesh_int(g.get("slowGenSeconds"), 20),
        "cpuCapPercent": _mesh_int(g.get("cpuCapPercent"), 50),
        "killEnabled": g.get("killEnabled") is not False,
        "expendable": expendable,
        "warnAfterSeconds": _mesh_int(g.get("warnAfterSeconds"), 6),
        "killGraceSeconds": _mesh_int(g.get("killGraceSeconds"), 15),
    }


def intel_settings(cfg):
    # Threat Intelligence Center (Build 15, #165): absent-block-
    # safe like the rest. Defaults: enabled, every keyless source
    # on, an empty watchlist, this node collects for itself (no
    # collector peer named), no silent local fallback, a 6-hour
    # collection cadence (EPSS rides daily inside the engine).
    b = cfg.get("intel") if isinstance(cfg, dict) else None
    if not isinstance(b, dict):
        b = {}
    src = b.get("sources") if isinstance(b.get("sources"), dict) else {}
    watch = b.get("watchlist")
    if not isinstance(watch, list):
        watch = []
    watch = [str(x).strip() for x in watch if str(x).strip()][:40]
    return {
        "enabled": b.get("enabled") is not False,
        "sources": {
            "nvd": src.get("nvd") is not False,
            "kev": src.get("kev") is not False,
            "epss": src.get("epss") is not False,
            "cisa": src.get("cisa") is not False,
            "rss": src.get("rss") is not False,
        },
        "watchlist": watch,
        "collectorPeer": str(b.get("collectorPeer") or "").strip(),
        "fallbackLocal": b.get("fallbackLocal") is True,
        "cadenceHours": _mesh_int(b.get("cadenceHours"), 6),
    }


def wire_settings(cfg):
    # The news wire (0.2.0e Build 7, #188): absent-block-safe
    # like the rest. Defaults are the prep's: enabled, a
    # 30-minute fetch interval, 7 days of retention.
    b = cfg.get("wire") if isinstance(cfg, dict) else None
    if not isinstance(b, dict):
        b = {}
    # Operator feeds (Build 3, #210): name + URL pairs the
    # operator added, validated at add time (see /api/wire/feeds).
    # They live in navi.config.json, which is NOT part of the
    # Sync set — feeds are per-seat until the operator says
    # otherwise, and the build report says so out loud.
    feeds = []
    for f in b.get("feeds") or []:
        if not isinstance(f, dict):
            continue
        url = str(f.get("url") or "").strip()
        name = str(f.get("name") or "").strip()[:80]
        if not name or not (url.startswith("http://")
                            or url.startswith("https://")):
            continue
        fid = str(f.get("id") or "") or (
            "custom-" + hashlib.sha1(url.encode("utf-8")).hexdigest()[:10])
        feeds.append({"id": fid, "name": name, "url": url[:500]})
    return {
        "enabled": b.get("enabled") is not False,
        "intervalMinutes": max(5, min(240, _mesh_int(b.get("intervalMinutes"), 30))),
        "retentionDays": max(1, min(30, _mesh_int(b.get("retentionDays"), 7))),
        "feeds": feeds,
    }


def ctf_settings(cfg):
    # The CTF board (0.2.0e Build 7, #189): enabled by default,
    # an hourly fetch.
    b = cfg.get("ctf") if isinstance(cfg, dict) else None
    if not isinstance(b, dict):
        b = {}
    return {
        "enabled": b.get("enabled") is not False,
        "intervalMinutes": max(10, min(720, _mesh_int(b.get("intervalMinutes"), 60))),
    }


def bounties_settings(cfg):
    # The bounty board (0.2.0e Build 7, Scope Addition 5):
    # enabled by default, the Chaos index re-read every 6
    # hours - program lists move slowly.
    b = cfg.get("bounties") if isinstance(cfg, dict) else None
    if not isinstance(b, dict):
        b = {}
    return {
        "enabled": b.get("enabled") is not False,
        "intervalMinutes": max(30, min(2880, _mesh_int(b.get("intervalMinutes"), 360))),
    }


MESH_DEFAULT_EXTRA_PORTS = (8443, 2475)


def _mesh_listen_ports(raw, canonical):
    # Build 4 (#154): the Mesh door listens on several ports —
    # one door, one certificate, several numbers, because home
    # gateways and workplace egress rules do not all let the
    # canonical port through. The canonical port (the one the
    # pairing card carries) is always first and always on; the
    # shipped defaults add 8443 and 2475 when the operator has
    # not configured a set of their own. Every entry must be a
    # plain unprivileged port; junk is dropped, never served.
    canon = _mesh_int(canonical, 8765)
    if canon < 1 or canon > 65535:
        canon = 8765
    ports = []
    if isinstance(raw, (list, tuple)):
        for v in raw:
            p = _mesh_int(v, 0)
            if 1024 < p <= 65535 and p not in ports:
                ports.append(p)
    if not ports:
        ports = [canon] + [p for p in MESH_DEFAULT_EXTRA_PORTS
                           if p != canon]
    elif canon in ports:
        ports.remove(canon)
        ports.insert(0, canon)
    else:
        ports.insert(0, canon)
    return ports[:8]


def mesh_settings(cfg):
    # Absent-block-safe, like dhd_settings: an install whose config
    # predates the mesh block (or a field of it) reads as the
    # shipped defaults — absent = off.
    m = cfg.get("mesh") if isinstance(cfg, dict) else None
    if not isinstance(m, dict):
        m = {}
    borrow = m.get("borrow") if isinstance(m.get("borrow"), dict) else {}
    lend = m.get("lend") if isinstance(m.get("lend"), dict) else {}
    role = str(m.get("role") or "both").strip().lower()
    if role not in ("borrow", "lend", "both"):
        role = "both"
    peers = []
    raw_peers = borrow.get("peers")
    if isinstance(raw_peers, list):
        for p in raw_peers:
            peer = _mesh_peer(p)
            if peer is not None:
                peers.append(peer)
    return {
        "enabled": m.get("enabled") is True,
        "role": role,
        "nodeName": str(m.get("nodeName") or ""),
        "borrow": {
            # R1: borrowing defaults ON under the master switch.
            "enabled": borrow.get("enabled") is not False,
            "graceMs": _mesh_int(borrow.get("graceMs"), 2000),
            "peers": peers,
        },
        "lend": {
            "enabled": lend.get("enabled") is True,
            "maxSharePercent": _mesh_int(lend.get("maxSharePercent"), 25),
            "maxGenerationsPerHour": _mesh_int(lend.get("maxGenerationsPerHour"), 20),
            "perPeerPerHour": _mesh_int(lend.get("perPeerPerHour"), 10),
            "idleOnly": lend.get("idleOnly") is not False,
            "idleAfterSeconds": _mesh_int(lend.get("idleAfterSeconds"), 60),
        },
        "port": _mesh_int(m.get("port"), 8765),
        "listenPorts": _mesh_listen_ports(m.get("listenPorts"),
                                          _mesh_int(m.get("port"), 8765)),
        "nodeId": str(m.get("nodeId") or ""),
        "ledger": m.get("ledger") is not False,
        "ledgerInBackup": m.get("ledgerInBackup") is True,
        # Build 6 (#143 ask 3): whether the first-run certificate
        # notice has been shown with a pairing on this machine.
        # The notice is part of the pairing act itself (see
        # mesh_mark_cert_notice), never a separate gate.
        "certNoticeSeen": m.get("certNoticeSeen") is True,
        # Build 4 (#154): this machine's own operator-declared
        # public endpoint (host:port), or "" — typed on the Mesh
        # page, carried on this machine's card, never harvested.
        "publicEndpoint": _mesh_endpoint_clean(m.get("publicEndpoint")),
    }


# -- Guardian service global (Build 14, #156) -------------------------
# The engine runs whenever the bridge runs (the sampler is started
# in __main__ beside the other background services, and re-checks
# the enabled setting every tick), including headless: protecting
# Navi's headroom matters most when nobody is watching the
# dashboard. Generation activity is a module-level counter the
# llamafile_chat wrapper below maintains, so Guardian can judge
# pressure one level stricter exactly while she is generating.
_GENERATION_LOCK = threading.Lock()
_GENERATION_ACTIVE = {"count": 0}

# -- Her own vitals (v0.4.6e, #187) -----------------------------------
# Spartan0311 caught Navi inventing numbers about herself —
# "gen speed is good, at 110 words per minute… We've been
# running stress tests" — with no measurement behind any of
# it (her real pace on that box was ~6.7 tok/s). She cannot
# feel her own speed; the only honest numbers are ones the
# bridge measures. So the bridge times every local model
# request it makes (the llamafile_chat wrapper records here;
# the impl stashes the server's own timings/usage alongside),
# and model_vitals_block() puts the recent figures in her
# context, beside the Guardian brief. VITALS_TEACHING is the
# other half of the fix: absent a measurement, she says so
# plainly — she never invents a speed, a benchmark, or a test
# she did not run.
_MODEL_VITALS = {"count": 0, "last": None, "recent": []}
_LAST_GEN_SERVER = {}

VITALS_TEACHING = (
    "Your own speed: you cannot feel how fast you generate, "
    "how long a request took, or how loaded the machine is "
    "under you — you have no senses pointed at yourself. The "
    "bridge measures your generation and, when it has, shows "
    "you the figures in a [Your measured vitals] block. Answer "
    "questions about your speed or performance ONLY from those "
    "measurements; when none are shown, say plainly that you "
    "have no measurement — never invent a number, a benchmark, "
    "a stress test, or a result you did not see.")


def _record_model_vitals(duration_s):
    # Called by the llamafile_chat wrapper after each
    # SUCCESSFUL local generation — a failed request never
    # replaces the last good measurement. tok/s and
    # time-to-first-token come from
    # the server's own timings when it reports them (llama.cpp
    # servers do); the bridge's wall-clock duration is always
    # real. Figures are rounded for speech, not for show.
    srv = dict(_LAST_GEN_SERVER)
    timings = srv.get("timings") or {}
    usage = srv.get("usage") or {}
    tokens = usage.get("completion_tokens")
    if tokens is None:
        tokens = timings.get("predicted_n")
    tok_s = None
    if timings.get("predicted_n") and timings.get("predicted_ms"):
        tok_s = timings["predicted_n"] / (timings["predicted_ms"] / 1000.0)
    elif tokens and duration_s:
        tok_s = tokens / duration_s
    ttft = None
    if timings.get("prompt_ms") is not None:
        ttft = timings["prompt_ms"] / 1000.0
    with _GENERATION_LOCK:
        _MODEL_VITALS["count"] += 1
        if tok_s:
            # v0.4.9e (#248): the recent pace — the last few
            # measured speeds, so "sustained slow" is a fact
            # with a sample behind it, not one bad second.
            _MODEL_VITALS["recent"] = (
                _MODEL_VITALS["recent"] + [round(tok_s, 1)])[-6:]
        if tokens or tok_s:
            _MODEL_VITALS["last"] = {
                "at": time.time(),
                "duration_s": round(duration_s, 2),
                "tokens": tokens,
                "tok_s": round(tok_s, 1) if tok_s else None,
                "ttft_s": round(ttft, 2) if ttft is not None else None,
            }


def model_vitals_block():
    """The recent measured figures, as context for her — or
    "" when the bridge has measured nothing yet (the teaching
    still rides; the absence IS the honest state)."""
    with _GENERATION_LOCK:
        last = dict(_MODEL_VITALS["last"] or {})
        count = _MODEL_VITALS["count"]
    if not last:
        return ""
    bits = []
    if last.get("tokens"):
        bits.append(str(last["tokens"]) + " tokens")
    if last.get("tok_s"):
        bits.append("at " + str(last["tok_s"]) + " tokens/sec")
    out = "[Your measured vitals — the bridge's stopwatch, this machine]\n"
    out += "Your last generation: " + (
        ", ".join(bits) if bits else "completed")
    if last.get("ttft_s") is not None:
        out += "; first token after " + str(last["ttft_s"]) + "s"
    out += ("; the whole request took " + str(last["duration_s"])
            + "s. Generations measured since this bridge "
              "started: " + str(count) + ".")
    with _GENERATION_LOCK:
        recent = list(_MODEL_VITALS["recent"])
    if len(recent) >= 3:
        out += ("\nRecent pace: your last " + str(len(recent))
                + " generations averaged "
                + str(round(sum(recent) / len(recent), 1))
                + " tokens/sec (slowest " + str(min(recent))
                + ", fastest " + str(max(recent)) + ").")
    return out


# v0.4.9e (#248): sluggish is a measured state, not a mood —
# a recent average under SLOW_TOK_S across at least three
# measured generations. (Her field-measured pace on a potato
# was ~6.7 tok/s; under roughly 8, conversation visibly lags.)
SLOW_TOK_S = 8.0
SLOW_MIN_SAMPLES = 3


def vitals_sluggish():
    """(True, avg) when the recent measured pace is
    sustained-slow on this hardware; (False, avg-or-None)
    otherwise. The judgment she speaks from — the teaching
    decides what she does with it."""
    with _GENERATION_LOCK:
        recent = list(_MODEL_VITALS["recent"])
    if len(recent) < SLOW_MIN_SAMPLES:
        return (False, None)
    avg = round(sum(recent) / len(recent), 1)
    return (avg < SLOW_TOK_S, avg)


def _generation_is_active():
    with _GENERATION_LOCK:
        return _GENERATION_ACTIVE["count"] > 0


def _guardian_llamafile_pids():
    # The bridge's own handle on the llamafile process, when this
    # bridge started her; when she was started by a launcher the
    # handle is empty and Guardian's own /proc scan (for
    # "navi.llamafile" in the cmdline) is the fallback.
    proc = globals().get("_llamafile_proc")
    try:
        if proc is not None and proc.poll() is None:
            return [proc.pid]
    except Exception:
        pass
    return []


def _guardian_mesh():
    # The Mesh facts Guardian's state carries: (enabled, peers).
    try:
        _ms = mesh_settings(load_config())
        return (bool(_ms["enabled"]), len(_ms["borrow"]["peers"]))
    except Exception:
        return (False, 0)


class _GuardianUnavailable:
    # The inert stand-in used only when guardian.py is missing
    # from the tree (see the import note at the top): state reads
    # as a disabled Guardian with an empty world rather than
    # crashing a route, and every action is a no-op.
    def start(self):
        return False

    def stop(self):
        pass

    def note_generation(self, duration_s):
        pass

    def context_block(self):
        return "Guardian: not installed in this tree."

    def state(self):
        _s = guardian_settings(load_config())
        return {
            "enabled": False, "pressure": "ok", "freeMb": 0,
            "reserveMb": _s["reserveMb"], "llamafileRssMb": 0,
            "generating": False, "mesh": {"enabled": False, "peers": 0},
            "top": [], "offenders": [], "expendable": list(_s["expendable"]),
            "settings": _s, "log": [],
        }


if guardian_mod is not None:
    GUARDIAN = guardian_mod.Guardian(
        ROOT,
        settings_getter=lambda: guardian_settings(load_config()),
        callbacks={
            "own_pid": os.getpid(),
            "llamafile_pids": _guardian_llamafile_pids,
            "is_generating": _generation_is_active,
        },
        mesh_getter=_guardian_mesh,
    )
else:
    GUARDIAN = _GuardianUnavailable()


# -- Threat Intelligence Center engine (Build 15, #165) ------------
# intel.py beside the bridge, imported like guardian.py: an
# install missing the module runs with the Center dark rather
# than failing to boot. The engine owns the store, the pullers,
# the matcher and the digest; the bridge owns config, the Mesh
# door it travels through, and the chat seams it surfaces in.
try:
    import intel as intel_mod
except Exception:
    intel_mod = None


class _IntelUnavailable:
    # The Guardian's inert shape: every surface answers
    # honestly-empty when intel.py is absent.
    def start(self):
        pass

    def state(self):
        return {"ok": False, "error": "intel.py is missing from this install — The Wire is unavailable.", "sources": [], "queue": [], "advisories": [], "watchlist": []}

    def brief_block(self, for_share=False):
        return ""

    def message_context(self, _text):
        return ""

    def case_context(self, _terms):
        return ""

    def case_matches(self, _terms):
        return []

    def lookup(self, _query):
        return "The Wire is unavailable on this install (intel.py missing)."

    def build_digest(self):
        return {}

    def shared_digest(self):
        return {}

    def import_digest(self, _digest):
        return {"ok": False, "error": "intel.py missing"}

    def refresh_now(self):
        return {"ok": False, "error": "intel.py missing"}

    def compute_matches(self):
        return 0

    def item(self, _cve):
        return None


def _intel_identity():
    cfg = load_config()
    ms = mesh_settings(cfg)
    return {"nodeId": ms["nodeId"], "name": ms["nodeName"] or socket.gethostname()}


def _intel_peers():
    try:
        cfg = load_config()
        return [{"nodeId": p.get("nodeId"), "name": p.get("name")}
                for p in mesh_settings(cfg)["borrow"]["peers"]]
    except Exception:
        return []


def _intel_case_terms(cfg, slug):
    # v0.3.0e Build 2 (#199, prep Part 2): the case-terms source
    # is the case's REGISTERS now — observables first (the IOC
    # register is the watchlist-shaped source: exact values, not
    # words scraped from prose), then entity names and aliases,
    # then the case's tags. The Build 15 scrape of case.md +
    # finding titles is retired: it fed the matcher every word
    # of the manifest, stopwords and all, and called them case
    # terms. A term here is a value an investigator actually
    # wrote down. Multi-word entity names also contribute their
    # words (the matcher is word-shaped), and observable values
    # ride whole (an IP or domain matches as itself).
    terms = []

    def _add(value):
        v = str(value or "").strip().lower()
        if len(v) >= 3 and v not in terms:
            terms.append(v)

    try:
        for ob in _case_register(slug, "observables"):
            _add(ob["heading"])
        for en in _case_register(slug, "entities"):
            _add(en["heading"])
            for alias in str(en["fields"].get("aliases") or "").split(","):
                _add(alias)
            for word in re.findall(r"[a-z0-9][a-z0-9._-]{2,}", str(en["heading"]).lower()):
                _add(word)
        meta = _case_manifest(slug) or {}
        for tag in str(meta.get("tags") or "").split(","):
            _add(tag)
    except Exception:
        return []
    return [{"term": t} for t in terms]


def _intel_case_terms_all():
    cfg = load_config()
    out = []
    try:
        for c in cases_list_payload()["cases"]:
            out.extend(_intel_case_terms(cfg, c["slug"]))
    except Exception:
        pass
    return out


def _intel_fetch_digest(peer_id):
    # 0.2.0e Build 1 (#143): returns (digest, "") on success or
    # (None, reason-in-words) \u2014 a refused or failed fetch names
    # the check that refused it, and the Center records the
    # reason on its Mesh source line instead of a bare
    # "collector unreachable".
    cfg = load_config()
    peer = None
    for p in mesh_settings(cfg)["borrow"]["peers"]:
        if p.get("nodeId") == peer_id:
            peer = p
            break
    if not peer:
        return None, "the named collector is not a paired peer on this machine"
    try:
        code, raw = _mesh_request(peer, "GET", "/mesh/intel-digest", b"", "", 60)
    except _MeshFpError:
        return None, "the collector's certificate does not match its pairing card \u2014 its Mesh identity has changed; re-pair from a fresh card"
    except _MeshTlsError:
        return None, "the secure (TLS) handshake with the collector failed"
    except _MeshTimeout:
        return None, "the collector did not answer in time"
    except _MeshUnreachable:
        return None, "the collector could not be reached"
    except Exception:
        return None, "the collector could not be reached"
    if code in (401, 403):
        return None, "the collector refused this machine's signature \u2014 check the two machines' clocks (a Mesh signature lives for 120 seconds); if they agree, re-pair from fresh cards"
    if code != 200:
        return None, "the collector answered with HTTP " + str(code)
    try:
        parsed = json.loads(raw.decode("utf-8", "replace"))
    except Exception:
        return None, "the collector's answer was not a readable digest"
    if not isinstance(parsed, dict):
        return None, "the collector's answer was not a readable digest"
    return parsed, ""


if intel_mod is not None:
    INTEL = intel_mod.Intel(
        ROOT,
        settings_getter=lambda: intel_settings(load_config()),
        callbacks={
            "identity": _intel_identity,
            "peers": _intel_peers,
            "case_terms": _intel_case_terms_all,
            "fetch_digest": _intel_fetch_digest,
        },
    )
else:
    INTEL = _IntelUnavailable()


# -- Awareness engines (0.2.0e Build 7, #188 / #189 / Addition 5)
# wire.py, ctf.py and bounties.py beside the bridge, imported
# like intel.py: an install missing a module runs with that
# family dark rather than failing to boot. The bridge owns
# config and the chat seam; the engines own their stores.
try:
    import wire as wire_mod
except Exception:
    wire_mod = None
try:
    import ctf as ctf_mod
except Exception:
    ctf_mod = None
try:
    import bounties as bounties_mod
except Exception:
    bounties_mod = None
try:
    import rowfeed as rowfeed_mod
except Exception:
    rowfeed_mod = None


class _WireUnavailable:
    def start(self):
        pass

    def state(self):
        return {"ok": False, "error": "wire.py is missing from this install — the news wire is unavailable.", "items": [], "sources": []}

    def brief_block(self):
        return ""

    def refresh_now(self):
        return {"ok": False, "error": "wire.py missing"}

    def read_article(self, _link):
        return {"ok": False, "reason": "unavailable", "sentence": "The reader is unavailable on this install (wire.py missing)."}


class _CtfUnavailable:
    def start(self):
        pass

    def state(self):
        return {"ok": False, "error": "ctf.py is missing from this install — the CTF board is unavailable.", "running": [], "upcoming": []}

    def brief_block(self):
        return ""

    def refresh_now(self):
        return {"ok": False, "error": "ctf.py missing"}


class _BountiesUnavailable:
    def start(self):
        pass

    def state(self):
        return {"ok": False, "error": "bounties.py is missing from this install — the bounty board is unavailable.", "programs": [], "reports": []}

    def brief_block(self):
        return ""

    def refresh_now(self):
        return {"ok": False, "error": "bounties.py missing"}


if wire_mod is not None:
    WIRE = wire_mod.Wire(ROOT, settings_getter=lambda: wire_settings(load_config()))
else:
    WIRE = _WireUnavailable()

if ctf_mod is not None:
    CTF = ctf_mod.Ctf(ROOT, settings_getter=lambda: ctf_settings(load_config()))
else:
    CTF = _CtfUnavailable()

if bounties_mod is not None:
    BOUNTIES = bounties_mod.Bounties(ROOT, settings_getter=lambda: bounties_settings(load_config()))
else:
    BOUNTIES = _BountiesUnavailable()


def _awareness_brief_block():
    # The composed situation briefing: the wire's sections, the
    # CTF board, the bounty board — one block, so "what's the
    # sitch" reads whole. Each engine's block carries its own
    # staleness; a family with nothing contributes nothing.
    parts = []
    for engine in (WIRE, CTF, BOUNTIES):
        try:
            blk = engine.brief_block()
        except Exception:
            blk = ""
        if blk:
            parts.append(blk)
    return "\n\n".join(parts)


# -- Node identity (spec section 4.2) -------------------------------
# An Ed25519 keypair under config/mesh/, made with the openssl
# binary — stdlib Python has no Ed25519, and openssl is already a
# DHD prerequisite. The private key never leaves this machine: it
# appears in no payload, no card, and no backup item.

def mesh_openssl_missing_error():
    return (
        "Apex | Mesh needs the openssl program to create this machine's node key, "
        "and openssl was not found on this machine. Install openssl (the same "
        "prerequisite DHD uses) and try again."
    )


def _mesh_pub_der(pub_pem):
    # The DER bytes of a PEM public key, via openssl — the nodeId
    # is the SHA-256 of exactly these bytes.
    fd, path = tempfile.mkstemp(suffix=".pem")
    try:
        with os.fdopen(fd, "w") as f:
            f.write(pub_pem)
        r = subprocess.run(
            ["openssl", "pkey", "-pubin", "-in", path, "-outform", "DER"],
            capture_output=True, timeout=30,
        )
        if r.returncode != 0:
            return b""
        return r.stdout
    finally:
        try:
            os.unlink(path)
        except Exception:
            pass


def mesh_node_id():
    if os.path.exists(MESH_PUB_PATH):
        der = _mesh_pub_der(read_file(MESH_PUB_PATH))
        if der:
            return hashlib.sha256(der).hexdigest()
    return ""


def mesh_ensure_identity(cfg):
    # Generates the keypair on first Mesh enable and records the
    # nodeId in the config. Returns (ok, error-in-plain-words).
    if shutil.which("openssl") is None:
        return False, mesh_openssl_missing_error()
    try:
        os.makedirs(MESH_DIR, exist_ok=True)
        if not (os.path.exists(MESH_KEY_PATH) and os.path.exists(MESH_PUB_PATH)):
            r = subprocess.run(
                ["openssl", "genpkey", "-algorithm", "ED25519", "-out", MESH_KEY_PATH],
                capture_output=True, text=True, timeout=60,
            )
            if r.returncode != 0:
                return False, "openssl could not create the Mesh node key: " + (r.stderr or "").strip()[:200]
            r = subprocess.run(
                ["openssl", "pkey", "-in", MESH_KEY_PATH, "-pubout", "-out", MESH_PUB_PATH],
                capture_output=True, text=True, timeout=60,
            )
            if r.returncode != 0:
                return False, "openssl could not write the Mesh node public key: " + (r.stderr or "").strip()[:200]
            try:
                os.chmod(MESH_KEY_PATH, 0o600)
            except Exception:
                pass
        node_id = mesh_node_id()
        if not node_id:
            return False, "The Mesh node key exists but could not be read back — check config/mesh/ on this machine."
        block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
        block = dict(block)
        if block.get("nodeId") != node_id:
            block["nodeId"] = node_id
            cfg["mesh"] = block
            save_config(cfg)
        return True, ""
    except Exception as e:
        return False, "Mesh identity could not be set up: " + str(e)


def mesh_sign(data):
    # Ed25519 signature (base64) over data, with this node's key.
    fd, path = tempfile.mkstemp()
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(data)
        r = subprocess.run(
            ["openssl", "pkeyutl", "-sign", "-inkey", MESH_KEY_PATH, "-rawin", "-in", path],
            capture_output=True, timeout=30,
        )
        if r.returncode != 0:
            raise RuntimeError("openssl could not sign the Mesh request")
        return base64.b64encode(r.stdout).decode()
    finally:
        try:
            os.unlink(path)
        except Exception:
            pass


def mesh_verify(pub_pem, data, sig_b64):
    # Verify an Ed25519 signature against a peer's pinned public
    # key. Any doubt at all reads as failure.
    try:
        sig = base64.b64decode(sig_b64 or "")
    except Exception:
        return False
    paths = []
    try:
        for blob, suffix in ((pub_pem.encode(), ".pem"), (data, ".bin"), (sig, ".sig")):
            fd, path = tempfile.mkstemp(suffix=suffix)
            with os.fdopen(fd, "wb") as f:
                f.write(blob)
            paths.append(path)
        r = subprocess.run(
            ["openssl", "pkeyutl", "-verify", "-pubin", "-inkey", paths[0],
             "-rawin", "-in", paths[1], "-sigfile", paths[2]],
            capture_output=True, timeout=30,
        )
        return r.returncode == 0
    except Exception:
        return False
    finally:
        for path in paths:
            try:
                os.unlink(path)
            except Exception:
                pass


def mesh_sig_payload(request_id, ts, body):
    # The signed string (spec section 4.3): requestId, timestamp,
    # and the body's SHA-256, one per line.
    return (
        str(request_id) + "\n" + str(ts) + "\n"
        + hashlib.sha256(body or b"").hexdigest()
    ).encode()


def mesh_signed_headers(request_id, body):
    ts = str(int(time.time()))
    return {
        "X-Mesh-Node": mesh_node_id(),
        "X-Mesh-Ts": ts,
        "X-Mesh-Sig": mesh_sign(mesh_sig_payload(request_id, ts, body)),
        "Content-Type": "application/json",
    }


def mesh_authenticate(handler, body, request_id):
    # Verifies a signed Mesh request against the paired-peer
    # list, inside the door's knock accounting (Build 4, #154):
    # a failure is a knock against the source, a success clears
    # the source's slate. The answer the caller gives is
    # unchanged — a bare "unauthorized"; WHY it failed never
    # leaves here (spec section 4.3: no detail leakage). Only
    # the door's own handlers call this, so every call is a
    # knock at the Mesh door by construction.
    peer = _mesh_authenticate(handler, body, request_id)
    try:
        ip = handler.client_address[0] if handler.client_address else ""
    except Exception:
        ip = ""
    if ip:
        if peer is not None:
            mesh_door_success(ip)
        else:
            has_headers = bool(handler.headers.get("X-Mesh-Node")) if getattr(handler, "headers", None) else False
            mesh_door_knock(ip, "signature" if has_headers else "unsigned")
    return peer


def _mesh_authenticate(handler, body, request_id):
    # Verifies a signed Mesh request against the paired-peer list.
    # Returns the peer record, or None — the caller answers a bare
    # "unauthorized" either way; WHY it failed never leaves here
    # (spec section 4.3: no detail leakage).
    # Build 7 (#87): pairing is the whole consent on this side —
    # any peer on the paired list with a valid key and signature
    # is authorized. The peer record's canBorrow flag used to be
    # required here too, which made one flag do two opposite jobs:
    # a lender whose stored flag for the requester was off refused
    # even a health probe as "unauthorized", invisibly. canBorrow
    # now means one thing only, enforced on the BORROWING side in
    # mesh_try_borrow: this machine will not borrow from a peer
    # whose flag is off.
    node = handler.headers.get("X-Mesh-Node") or ""
    ts = handler.headers.get("X-Mesh-Ts") or ""
    sig = handler.headers.get("X-Mesh-Sig") or ""
    if not node or not ts or not sig:
        return None
    try:
        ts_i = int(ts)
    except Exception:
        return None
    if abs(time.time() - ts_i) > 120:
        return None
    ms = mesh_settings(load_config())
    peer = None
    for p in ms["borrow"]["peers"]:
        if p["nodeId"] == node:
            peer = p
    if peer is None or not peer["publicKeyPem"]:
        return None
    if not mesh_verify(peer["publicKeyPem"], mesh_sig_payload(request_id, ts, body), sig):
        return None
    return peer


# -- Mesh certificate (spec section 4.1) ----------------------------
# Mesh carries its OWN self-signed certificate under config/mesh/,
# made the same way DHD makes its — always (build .86): the
# dedicated Mesh door has its own identity in every configuration,
# so the fingerprint a peer pins from the card is the Mesh cert's
# whether or not DHD is on. The borrower PINS the certificate's
# SHA-256 fingerprint from the peer card — that pin, not a CA, is
# what authenticates the door.

def mesh_active_cert():
    # (cert_path, key_path) this node serves Mesh on: the Mesh
    # self-signed pair under config/mesh/, in all cases.
    return MESH_CERT_PATH, MESH_CERT_KEY_PATH


def mesh_ensure_cert():
    cert_path, _key_path = mesh_active_cert()
    if cert_path == MESH_CERT_PATH and not os.path.exists(MESH_CERT_PATH):
        if shutil.which("openssl") is None:
            return False, mesh_openssl_missing_error()
        os.makedirs(MESH_DIR, exist_ok=True)
        sans = ["DNS:localhost", "IP:127.0.0.1"]
        try:
            ip = lan_ipv4()
            if ip and ip != "127.0.0.1":
                sans.append("IP:" + ip)
        except Exception:
            pass
        r = subprocess.run(
            ["openssl", "req", "-x509", "-newkey", "rsa:2048",
             "-keyout", MESH_CERT_KEY_PATH, "-out", MESH_CERT_PATH,
             "-days", "365", "-nodes", "-subj", "/CN=Apex Mesh",
             "-addext", "subjectAltName=" + ",".join(sans)],
            capture_output=True, text=True, timeout=60,
        )
        if r.returncode != 0:
            return False, "openssl could not generate the Mesh certificate: " + (r.stderr or "").strip()[:200]
    return True, ""


def mesh_cert_fingerprint():
    cert_path, _key_path = mesh_active_cert()
    if not os.path.exists(cert_path):
        ok, _err = mesh_ensure_cert()
        if not ok:
            return ""
    try:
        r = subprocess.run(
            ["openssl", "x509", "-in", cert_path, "-outform", "DER"],
            capture_output=True, timeout=30,
        )
        if r.returncode != 0 or not r.stdout:
            return ""
        return hashlib.sha256(r.stdout).hexdigest()
    except Exception:
        return ""


# -- Peer card (spec section 3.1) -----------------------------------

def _interface_ipv4s():
    # Every non-loopback IPv4 address on this machine's network
    # interfaces, via the SIOCGIFADDR ioctl (stdlib only; Linux is
    # the bridge's platform). Best-effort by construction: any
    # failure — no fcntl, an interface that will not answer —
    # simply contributes nothing, and the caller keeps the
    # default-route address it already has. Card plumbing only:
    # these addresses are never rendered or logged anywhere.
    out = []
    try:
        import fcntl
        import struct
        for _idx, name in socket.if_nameindex():
            try:
                s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
                try:
                    packed = fcntl.ioctl(s.fileno(), 0x8915, struct.pack("256s", name.encode()[:15]))
                finally:
                    s.close()
                ip = socket.inet_ntoa(packed[20:24])
            except Exception:
                continue
            if ip and not ip.startswith("127.") and ip not in out:
                out.append(ip)
    except Exception:
        pass
    return out


# Interface names that are never a machine's reachable LAN
# face, whatever address they hold: container bridges and
# their veth tails, VM bridges, tunnels and VPN adapters,
# overlay plumbing. The card filter decides by INTERFACE, not
# by address range (Build 4, #143): a 10.x address on the
# physical NIC is a legitimate LAN address, and a 192.168.x
# address on a container bridge is junk — the field failure
# behind this build was a card carrying a Docker bridge
# gateway that answered as the probing machine itself.
_MESH_VIRTUAL_IFACE_PREFIXES = (
    "docker", "veth", "virbr", "tun", "tap", "wg", "br-",
    "dummy", "ifb", "cali", "flannel", "cni", "kube", "lxdbr",
    "podman", "nerdctl", "tailscale",
)


def _mesh_filter_physical(facts):
    # The card generator's filter, pure over interface facts —
    # [{name, ip, isBridge}] in, the surviving facts out, in
    # order. An interface is excluded on any strong virtual
    # signal: it is loopback, its name is known virtual
    # plumbing (container bridges and their veth tails, VM
    # bridges, tunnels, VPN adapters), or the kernel marks it
    # a bridge. What is NOT a signal: the /sys device link.
    # Physical hardware has one, but so does nothing else —
    # virtio NICs, veth tails, and the user-mode networking
    # face of a sandboxed VM (this build machine's own host0,
    # its only way home) all lack it, and a rule requiring it
    # refuses honest cards on exactly the virtualized machines
    # Apex runs on. Interface type is judged by the interface's
    # own marks, never by its address range.
    out = []
    seen = set()
    for f in facts or []:
        name = str(f.get("name") or "").lower()
        ip = str(f.get("ip") or "")
        if not ip or ip.startswith("127.") or name == "lo":
            continue
        if any(name.startswith(p) for p in _MESH_VIRTUAL_IFACE_PREFIXES):
            continue
        if f.get("isBridge"):
            continue
        if ip in seen:
            continue
        seen.add(ip)
        out.append(f)
    return out


def _interface_facts():
    # Every interface's IPv4 with the facts the card filter
    # judges: its name and whether the kernel marks it a
    # bridge. Same SIOCGIFADDR walk as _interface_ipv4s, plus
    # the /sys read. Best-effort throughout.
    facts = []
    try:
        import fcntl
        import struct
        for _idx, name in socket.if_nameindex():
            ip = ""
            try:
                s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
                try:
                    packed = fcntl.ioctl(s.fileno(), 0x8915, struct.pack("256s", name.encode()[:15]))
                finally:
                    s.close()
                ip = socket.inet_ntoa(packed[20:24])
            except Exception:
                ip = ""
            is_bridge = False
            try:
                is_bridge = os.path.isdir(os.path.join("/sys/class/net", name, "bridge"))
            except Exception:
                is_bridge = False
            facts.append({"name": name, "ip": ip, "isBridge": is_bridge})
    except Exception:
        pass
    return facts


def _physical_interface_ipv4s():
    # The addresses a pairing card may honestly carry: IPv4s
    # bound to physical interfaces only — no loopback, no
    # container bridge, no VPN tail (Build 4, #143).
    return [f["ip"] for f in _mesh_filter_physical(_interface_facts())]


def _mesh_card_addresses(cfg, ms):
    # (addresses, refusal): the card's address set for this
    # machine — physical-LAN addresses, the default route's
    # address first when it is one of them, then the operator-
    # declared public endpoint as its own host:port entry.
    # When nothing plausible exists at all, addresses is empty
    # and refusal is the sentence the mint answers with: a card
    # carrying junk — or nothing — is how the week of dark Mesh
    # happened, and the generator now refuses instead.
    ips = _physical_interface_ipv4s()
    try:
        lan = lan_ipv4()
    except Exception:
        lan = ""
    if lan and lan in ips:
        ips = [lan] + [ip for ip in ips if ip != lan]
    addresses = list(ips)
    endpoint = ms["publicEndpoint"]
    if endpoint and endpoint not in addresses:
        addresses.append(endpoint)
    if addresses:
        return addresses, ""
    return [], ("This machine has no address its Mesh card can honestly carry: every network "
                "interface it can see is a loopback, a container bridge, or a virtual adapter, "
                "and no public endpoint is declared. Connect the machine to a real network — "
                "Ethernet or Wi-Fi — or declare a public endpoint on the Mesh page, and the "
                "card can be minted.")


def mesh_card_refusal(cfg):
    # "" when this machine's card can be minted, or the refusal
    # sentence. The card download answers with it instead of a
    # picture minted from junk.
    return _mesh_card_addresses(cfg, mesh_settings(cfg))[1]


def mesh_card(cfg):
    # This node's card data — the payload the PNG pairing picture
    # is minted from (build_mesh_card_png). The operator carries
    # the PICTURE to their other machine by hand; since Build 9
    # (#90) the card never leaves as copyable text. Names and
    # addresses come from config and this machine only (R2 —
    # nothing baked in).
    ms = mesh_settings(cfg)
    # Build 4 (#143): the card's addresses are the machine's
    # physical-LAN addresses (see _mesh_card_addresses) plus the
    # operator-declared public endpoint when one is set. What
    # the generator no longer does, each of which stranded a
    # real pair: harvest every interface (container bridges and
    # VPN tails answered probes as the probing machine itself),
    # append the DHD domain (a web door's name is not a Mesh
    # route), and append loopback (this machine is not its own
    # peer). When nothing plausible exists the set is empty and
    # the mint refuses — see mesh_card_refusal.
    addresses, _refusal = _mesh_card_addresses(cfg, ms)
    name = ms["nodeName"] or socket.gethostname()
    return {
        "meshCard": 1,
        "name": name,
        "nodeId": mesh_node_id(),
        "publicKeyPem": read_file(MESH_PUB_PATH).strip(),
        "certFingerprint": mesh_cert_fingerprint(),
        "addresses": addresses,
        "port": ms["port"],
        "capabilities": mesh_capabilities(cfg),
        "publicEndpoint": ms["publicEndpoint"],
        "tunnelPublicKey": mesh_tunnel_public_key(),
    }

