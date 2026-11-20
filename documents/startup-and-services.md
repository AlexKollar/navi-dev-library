# Startup and services

## start.sh

Source: `start.sh` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

#!/usr/bin/env bash
# Starts Navi: llamafile server (launch mode), the bridge, and her
# window. Apex stands alone now: the front door is the Tauri
# shell binary (navi-apex-shell, built by the installer from shell/)
# opening her own window on the bridge's dashboard — no web browser
# is opened. The browser openers survive only as the fallback for an
# install where the shell was never built (no Rust toolchain).
# Launch command: ./navi.llamafile --server --v2 plus the runtime
# flags from the hardware profile in config/navi.config.json
# (HW_FLAGS below; the fluid floor when the config has no profile).
set -euo pipefail
cd "$(dirname "$0")"
PORT="${NAVI_LLAMAFILE_PORT:-8080}"
HW_FLAGS="$(python3 - <<'PY' 2>/dev/null || echo "--ctx-size 4096 --threads 4 --batch-size 512"
import json
try:
    with open("config/navi.config.json") as f:
        cfg = json.load(f)
    rt = cfg.get("runtime") or {}
    hp = (cfg.get("hardware_profile") or {}).get("settings") or {}
    ctx = int(rt.get("ctx_size") or hp.get("ctxSize") or 4096)
    thr = int(rt.get("threads") or hp.get("threads") or 4)
    bat = int(rt.get("batch_size") or hp.get("batchSize") or 512)
    gpu = int(rt.get("gpu_layers") if rt.get("gpu_layers") is not None else (hp.get("gpuLayers") or 0))
    out = f"--ctx-size {ctx} --threads {thr} --batch-size {bat}"
    if gpu > 0:
        out += f" --n-gpu-layers {gpu}"
    print(out)
except Exception:
    print("--ctx-size 4096 --threads 4 --batch-size 512")
PY
)"


# Everything this launcher decides also lands in start.log beside
# the app. A menu/desktop launch has no terminal: the echoes go
# nowhere there, and a window that failed to open used to fail
# in total silence while the bridge ran fine. The log is how a
# menu launch explains itself after the fact.
LOG_FILE="start.log"
log_line() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE" 2>/dev/null || true; }
log_line "launch: Apex starting (llamafile port $PORT, bridge port 8675)"

# Headless server mode (Build 12, issue #142): --headless starts
# the services only — llamafile (check-first, as below) and the
# supervised bridge — and opens no window. It is the mode the
# navi-bridge.service user unit runs, and the mode an operator
# brings Apex back with over SSH after a reboot.
HEADLESS=""
for _arg in "$@"; do
  if [ "$_arg" = "--headless" ]; then HEADLESS=1; fi
done
if [ -n "$HEADLESS" ]; then
  log_line "launch mode: headless — services only, no window"
fi

if curl -s --max-time 2 "http://localhost:$PORT/" >/dev/null 2>&1; then
  # Check first, always: a llamafile server is already answering —
  # started at boot, by an earlier launch, or by hand — so leave it
  # alone and just run the app. Spawning a second copy would only
  # fail to bind the port and litter the log with a dead start.
  echo "navi.llamafile is already running on port $PORT — leaving it alone."
  log_line "llamafile: already running on port $PORT — left alone"
elif [ -x ./navi.llamafile ]; then
  echo "navi.llamafile is not running — starting it: ./navi.llamafile --server --v2 $HW_FLAGS (output: navi.llamafile.log) …"
  log_line "llamafile: not running — starting ./navi.llamafile --server --v2 $HW_FLAGS (hardware profile from config/navi.config.json)"
  # shellcheck disable=SC2086
  ./navi.llamafile --server --v2 $HW_FLAGS >> navi.llamafile.log 2>&1 &
  LLAMA_PID=$!
  for i in $(seq 1 120); do
    if curl -s --max-time 2 "http://localhost:$PORT/" >/dev/null 2>&1; then break; fi
    if ! kill -0 "$LLAMA_PID" 2>/dev/null; then
      echo "navi.llamafile exited during startup — last log lines:" >&2
      tail -n 5 navi.llamafile.log >&2 || true
      echo "The bridge can retry the start: open the UI and click Connect." >&2
      break
    fi
    sleep 1
  done
else
  echo "navi.llamafile is not in $(pwd) — the model server cannot start until the installer downloads it." >&2
fi

# The bridge runs on the voice venv's Python when that venv exists:
# it is a plain Python 3 with the (optional) speech engines
# installed, so her voice works; without the venv, the system
# python3 runs the same bridge, stdlib-only, voice unavailable
# until the engines are installed for it.
PYTHON="python3"
if [ -x "voice/venv/bin/python" ]; then
  PYTHON="voice/venv/bin/python"
fi

# The bridge exits with code 75 when it has been replaced — a newer
# bridge.py landed on disk, or a newer bridge claimed the port — so
# supervise it: relaunch on 75 and an update takes over on its own
# instead of leaving a zombie bridge answering with old code.
supervise_bridge() {
  while true; do
    code=0
    "$PYTHON" bridge.py || code=$?
    [ "$code" -eq 75 ] || break
    echo "Bridge replaced — relaunching the current build…"
    sleep 1
  done
}

BRIDGE_PORT="8675"
# Adoption (Build 12, issue #142): a Navi bridge already answering
# on the port — started at boot by navi-bridge.service, by an
# earlier headless start, or by hand — is ADOPTED, never doubled:
# a second bridge could only fight the first for the port. The
# desktop launch follows the same rule, so opening the app on a
# machine whose services are up uses them exactly as they are.
BRIDGE_ADOPTED=""
if curl -s --max-time 2 "http://localhost:$BRIDGE_PORT/api/status" 2>/dev/null | grep -q '"app": "Navi: Apex"'; then
  BRIDGE_ADOPTED=1
  echo "A Navi bridge is already answering on port $BRIDGE_PORT — adopting it, not starting a second one."
  log_line "bridge: already answering on port $BRIDGE_PORT — adopted, not doubled"
else
  echo "Starting bridge on http://localhost:$BRIDGE_PORT …"
  log_line "bridge: starting the supervised bridge on port $BRIDGE_PORT"
  supervise_bridge &
fi

if [ -n "$HEADLESS" ]; then
  # Verify the bridge is the thing answering before saying so —
  # the same discipline the window path below applies, kept in
  # the same log.
  for i in $(seq 1 40); do
    if curl -s --max-time 2 "http://localhost:$BRIDGE_PORT/api/status" 2>/dev/null | grep -q '"app": "Navi: Apex"'; then
      log_line "headless: bridge verified answering on port $BRIDGE_PORT — services up, no window"
      echo "Apex headless: services are up (llamafile port $PORT, bridge port $BRIDGE_PORT) — no window opened."
      break
    fi
    sleep 1
  done
  if [ -n "$BRIDGE_ADOPTED" ]; then
    # The adopted bridge is not our child, so `wait` has nothing
    # to hold: hold the foreground like a supervisor and step
    # aside when it stops answering.
    while curl -s --max-time 3 "http://localhost:$BRIDGE_PORT/api/status" 2>/dev/null | grep -q '"app": "Navi: Apex"'; do sleep 5; done
    log_line "headless: the adopted bridge stopped answering — exiting"
    exit 1
  fi
  wait
  exit 0
fi

# Her own window — the standalone Tauri shell — is the front door.
# It is single-instance (a second launch focuses the window that is
# already open) and it loads the dashboard from the bridge, so the
# bridge verification below still gates the open exactly as it did
# for the browser. Only an install where the shell was never built
# (no Rust toolchain at install time) falls back to a browser; the
# dashboard itself stays reachable at the same URL regardless.
# A shell binary is launched only when the stamp beside it covers
# the shell sources beside it — the same hash the installer writes
# after a successful build. A binary from an older build is a
# different app wearing this one's name. Verdicts: 0 =
# verified current, 1 = verified stale (or no binary), 2 = cannot
# verify (no hashing tool) — unverifiable launches as before,
# because a missing sha256sum must not strand the operator.
shell_sources_hash() {
  local dir="shell"
  # Same rule as the installer's copy of this hash: the shell icon
  # is optional — a clone install has no binary icon files, and
  # hashing must not fail over a cosmetic file. With the icon
  # present it is hashed in its usual place, unchanged.
  local files="Cargo.toml build.rs src/main.rs tauri.conf.json"
  if [ -f "$dir/icons/icon.png" ]; then files="$files icons/icon.png"; fi
  files="$files README.md"
  local f
  for f in $files; do
    if [ ! -f "$dir/$f" ]; then echo ""; return; fi
  done
  if command -v sha256sum >/dev/null 2>&1; then
    (cd "$dir" && sha256sum $files | sha256sum | awk '{print $1}')
  elif command -v shasum >/dev/null 2>&1; then
    (cd "$dir" && shasum -a 256 $files | shasum -a 256 | awk '{print $1}')
  else
    echo ""
  fi
}

shell_is_current() {
  [ -x "./navi-apex-shell" ] || return 1
  local h s
  h="$(shell_sources_hash)"
  if [ -z "$h" ]; then return 2; fi
  s=""
  if [ -f "./navi-apex-shell.stamp" ]; then s="$(cat ./navi-apex-shell.stamp)"; fi
  if [ "$s" = "$h" ]; then return 0; fi
  return 1
}

open_window() {
  if [ -x "./navi-apex-shell" ]; then
    local verdict=0
    shell_is_current || verdict=$?
    if [ "$verdict" -eq 1 ]; then
      log_line "window: the shell binary beside start.sh is STALE — built from older shell sources (its stamp does not cover shell/) — not launching it. Rebuild it (cd shell && cargo build --release) or re-run the installer."
      log_line "window: falling back to the browser until the shell is rebuilt"
      open_browser_fallback
      return 0
    fi
    if [ "$verdict" -eq 2 ]; then
      log_line "window: cannot verify the shell binary against its build stamp (no sha256sum/shasum) — launching it as-is"
    fi
      log_line "window: opening the standalone Apex window (Tauri shell)"
      ./navi-apex-shell >>"$LOG_FILE" 2>&1 &
    return 0
  fi
  log_line "window: no shell binary beside start.sh — falling back to the browser (build it with: cd shell && cargo build --release)"
  open_browser_fallback
}

open_browser_fallback() {
  local url="http://localhost:$BRIDGE_PORT"
  # Try each opener in turn and CHECK that it worked. An opener
  # that merely exists but fails — xdg-open with no default
  # browser registered answers "no method available" and exits
  # nonzero — must fall through to the next opener instead of
  # ending the attempt. That unchecked single xdg-open call was
  # the menu-launch failure: the bridge came up, the dashboard
  # was live, and the browser step died without a sound where
  # nobody could see it (a menu launch has no terminal).
  if command -v xdg-open >/dev/null 2>&1 && xdg-open "$url" >>"$LOG_FILE" 2>&1; then
    log_line "browser: opened $url with xdg-open"
    return 0
  fi
  if command -v gio >/dev/null 2>&1 && gio open "$url" >>"$LOG_FILE" 2>&1; then
    log_line "browser: opened $url with gio open"
    return 0
  fi
  if command -v sensible-browser >/dev/null 2>&1 && sensible-browser "$url" >>"$LOG_FILE" 2>&1; then
    log_line "browser: opened $url with sensible-browser"
    return 0
  fi
  if command -v open >/dev/null 2>&1 && open "$url" >>"$LOG_FILE" 2>&1; then
    log_line "browser: opened $url with open"
    return 0
  fi
  local b
  for b in firefox chromium chromium-browser google-chrome google-chrome-stable brave-browser microsoft-edge; do
    if command -v "$b" >/dev/null 2>&1; then
      "$b" "$url" >>"$LOG_FILE" 2>&1 &
      log_line "browser: launched $b with $url"
      return 0
    fi
  done
  log_line "browser: NO opener worked — the dashboard is live at $url; open it manually. (Tried xdg-open, gio, sensible-browser, open, and the common browser binaries; their output is above in this log.)"
  echo "Could not open a browser automatically — the dashboard is live at $url" >&2
  return 1
}

# Open her window once the process answering on the bridge port
# is a Navi bridge. Any build counts: when the answering bridge
# is an older or newer build than this launcher, its dashboard
# is still Navi — it carries the BRIDGE OUT OF DATE banner for
# exactly that case — and demanding an exact stamp match here
# stranded the operator with a running app and no window, the
# refusal explained only to a terminal a menu launch doesn't
# have. A page that is NOT a Navi bridge at all still blocks
# the open, and gets named — in the log as well as the terminal.
BRIDGE_STAMP="2026-11-09.91"
OPENED=""
for i in $(seq 1 40); do
  STATUS=$(curl -s --max-time 2 "http://localhost:$BRIDGE_PORT/api/status" 2>/dev/null || true)
  if printf '%s' "$STATUS" | grep -q '"app": "Navi: Apex"'; then
    OPENED=1
    STAMP_NEEDLE="\"bridge\": \"$BRIDGE_STAMP\""
    if printf '%s' "$STATUS" | grep -qF "$STAMP_NEEDLE"; then
      log_line "verify: this build's bridge ($BRIDGE_STAMP) is answering on port $BRIDGE_PORT"
    else
      log_line "verify: a Navi bridge of a different build than $BRIDGE_STAMP is answering on port $BRIDGE_PORT — opening it anyway; its dashboard flags the mismatch"
    fi
    break
  fi
  sleep 1
done
if [ -n "$OPENED" ]; then
  open_window || true
else
  TITLE=$(curl -s --max-time 2 "http://localhost:$BRIDGE_PORT/" 2>/dev/null | grep -oiE '<title[^>]*>[^<]*</title>' | head -1 | sed -E 's/<[^>]+>//g' || true)
  if [ -n "$TITLE" ]; then
    log_line "verify: port $BRIDGE_PORT is held by a different program (its page is titled "$TITLE") — window not opened"
    echo "Not opening the window: http://localhost:$BRIDGE_PORT is served by a different program (its page is titled \"$TITLE\")," >&2
    echo "not by the Apex bridge — her window would show that program's page, not this dashboard." >&2
    echo "Quit the program holding the port (find it with: lsof -i :$BRIDGE_PORT), then launch Apex again." >&2
  elif curl -s --max-time 2 http://localhost:8765/api/status 2>/dev/null | grep -q '"app": "Navi: Apex"'; then
    # A Navi bridge answering on the PREVIOUS package's port can only
    # be a bridge process from that older install still running (this
    # build's bridge binds $BRIDGE_PORT and nothing else). Name it
    # and say how to clear it instead of the generic shrug — this is
    # the exact shape of the port-change stranding: the old bridge
    # alive on 8765, the new launcher polling $BRIDGE_PORT in vain.
    log_line "verify: a previous install's Navi bridge is answering on the old port 8765 — window not opened"
    echo "Not opening the window: a Navi bridge from a previous install is still running on the old port, http://localhost:8765," >&2
    echo "and this build's bridge never took port $BRIDGE_PORT. Stop the old bridge (find it with: lsof -i :8765, or run:" >&2
    echo "pkill -f bridge.py), then launch Apex again." >&2
  else
    log_line "verify: no Navi bridge answered on port $BRIDGE_PORT — window not opened; see the bridge output in this folder"
    echo "The Navi bridge did not answer on http://localhost:$BRIDGE_PORT — not opening the window. See the bridge output above." >&2
  fi
fi
wait


## run-llamafile.sh

Source: `run-llamafile.sh` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

#!/usr/bin/env bash
# Apex — profile-driven llamafile launcher.
# Reads runtime settings from config/navi.config.json (written by
# the install-time hardware profiler; editable / re-detectable in
# Settings) and execs navi.llamafile with those flags. No profile
# in the config -> the fluid floor: 4096 / 4 threads / batch 512.
set -euo pipefail
cd "$(dirname "$0")"
FLAGS="$(python3 - <<'PY' 2>/dev/null || echo "--ctx-size 4096 --threads 4 --batch-size 512"
import json
try:
    with open("config/navi.config.json") as f:
        cfg = json.load(f)
    rt = cfg.get("runtime") or {}
    hp = (cfg.get("hardware_profile") or {}).get("settings") or {}
    ctx = int(rt.get("ctx_size") or hp.get("ctxSize") or 4096)
    thr = int(rt.get("threads") or hp.get("threads") or 4)
    bat = int(rt.get("batch_size") or hp.get("batchSize") or 512)
    gpu = int(rt.get("gpu_layers") if rt.get("gpu_layers") is not None else (hp.get("gpuLayers") or 0))
    out = f"--ctx-size {ctx} --threads {thr} --batch-size {bat}"
    if gpu > 0:
        out += f" --n-gpu-layers {gpu}"
    print(out)
except Exception:
    print("--ctx-size 4096 --threads 4 --batch-size 512")
PY
)"
# shellcheck disable=SC2086
exec ./navi.llamafile --server --v2 $FLAGS


## systemd/navi-bridge.service

Source: `systemd/navi-bridge.service` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

[Unit]
Description=Apex bridge — headless server mode (Powered by Navi)
After=network.target navi-llamafile.service
Wants=navi-llamafile.service

[Service]
ExecStart=__INSTALL_DIR__/start.sh --headless
Restart=on-failure
WorkingDirectory=__INSTALL_DIR__

[Install]
WantedBy=default.target


## systemd/navi-llamafile.service

Source: `systemd/navi-llamafile.service` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

[Unit]
Description=Navi llamafile server
After=network.target

[Service]
ExecStart=__INSTALL_DIR__/run-llamafile.sh
Restart=on-failure
WorkingDirectory=__INSTALL_DIR__

[Install]
WantedBy=default.target


## docker/entrypoint.sh

Source: `docker/entrypoint.sh` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

#!/usr/bin/env bash
# Apex container entrypoint (0.2.0e Build 7).
#
#   1. First boot only: seed the data volume from the image's
#      pristine tree, and say so — the volume's existence from
#      then on is how every later boot knows it is not the
#      first.
#   2. Run the first-boot step (docker/first_boot.py): on the
#      first boot it gives the node its container posture (DHD
#      on, Mesh on), mints its identity and its DHD access code
#      with the bridge's own machinery, and prints the code's
#      boxed banner ONCE. On later boots it prints none of it.
#   3. The model: downloaded once to the volume, on first run,
#      exactly as the installer does it; skipped when present.
#   4. Launch headless — llamafile + the supervised bridge —
#      in the foreground: start.sh --headless is the same mode
#      the systemd unit runs on bare metal.
set -euo pipefail

APP=/opt/navi-apex
DIST=/opt/navi-apex-dist
export APEX_CONTAINER=1

if [ ! -f "$APP/bridge.py" ]; then
  echo "Apex container: first boot — seeding the data volume from the image."
  mkdir -p "$APP"
  cp -a "$DIST/." "$APP/"
  export APEX_FIRST_BOOT=1
fi

cd "$APP"
python3 docker/first_boot.py

if [ ! -f ./navi.llamafile ]; then
  echo "Apex container: downloading the model (navi.llamafile) — first run only; it is kept in the data volume."
  curl -fL --retry 3 -o navi.llamafile.part \
    "https://huggingface.co/saintssec/navi.llamafile/resolve/main/navi.llamafile"
  chmod +x navi.llamafile.part
  mv navi.llamafile.part navi.llamafile
fi

exec ./start.sh --headless


## docker/first_boot.py

Source: `docker/first_boot.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

#!/usr/bin/env python3
# Apex container first boot (0.2.0e Build 7).
#
# Runs on EVERY container start, from the entrypoint, before
# anything serves. Almost everything it can do is gated on the
# volume's first boot (APEX_FIRST_BOOT, set by the entrypoint
# only when it just seeded the volume); on later boots it is a
# quiet no-op that re-prints nothing. What it does, in order,
# all of it through the bridge's OWN machinery — this file
# invents no second way of doing any of it:
#
#   1. The container posture (first boot): DHD enabled (the
#      no-domain first run — self-signed certificate, access
#      code), Mesh enabled, the node named from APEX_NODE_NAME
#      when given, the declared public endpoint seeded from
#      APEX_PUBLIC_ENDPOINT when given. Bare metal is untouched
#      by all of this: nothing here runs there, and the tree's
#      shipped defaults are exactly as shipped.
#   2. The node's identity (first boot): the Mesh keypair and
#      the door certificate, minted by mesh_ensure_identity /
#      mesh_ensure_cert into the volume. Born on the machine
#      that runs them; never baked in, never backed up.
#   3. The access code: on the first boot a FRESH code is
#      minted — the config's shipped placeholder is never
#      inherited by a container node, because a code everyone
#      can read is a vulnerability, not a feature. The mint is
#      announced in a boxed banner, printed ONCE, here, and the
#      same banner is kept in FIRST-BOOT.txt in the volume.
#      Later boots never re-print it; the deliberate doors are
#      the bridge's own: `python3 bridge.py --dhd-code`
#      (re-print) and `--dhd-code-reset` (a new code).
#   4. The federation invitation (0.2.0e Build 8, first boot):
#      an image built to join a fedinet carries the gateway's
#      card at docker/gateway-card.png. When it is present, it
#      is copied into the volume as the waiting invitation
#      (config/federation/invitation.png); the bridge imports
#      it at start — ADMISSION ONLY: the node is introduced to
#      the gateway, and nothing lends until the fedinet's
#      signed roster syncs. The card is public data (a pairing
#      card's facts + the gateway's signature); the gateway's
#      identity never leaves the gateway — the card is minted
#      THERE, never in the image build's audience. An image
#      carrying no card simply starts unfederated. (The tree's
#      docker/sample-gateway-card.png is a SAMPLE — its shape,
#      with sample facts — never wired as an invitation.)
import importlib.util
import os
import shutil
import socket
import sys

APP = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
os.chdir(APP)
sys.path.insert(0, APP)


def load_bridge():
    spec = importlib.util.spec_from_file_location(
        "bridgeFirstBoot", os.path.join(APP, "bridge.py"))
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    return mod


def banner(code, name, fingerprint, node_id):
    bar = "=" * 68
    lines = [
        bar,
        "  YOUR DHD ACCESS CODE — WRITE IT DOWN OR TAKE A PHOTO NOW",
        "",
        "      Code:  " + code,
        "      Node:  " + name + "  ·  node " + (node_id[:12] if node_id else "—"),
        "      Mesh fingerprint:  " + fingerprint,
        "",
        "  This code is the ONLY key to this node's DHD door on your",
        "  network, and the door is already on: browse to this",
        "  computer's address (https — the DHD door Docker publishes",
        "  on port 443), type the code once, name your device, and",
        "  you are in. Every device after the first waits for an",
        "  approval from one that is already trusted, and your Mesh",
        "  pairing card waits on the dashboard's Mesh page.",
        "",
        "  This is the only time the code is printed in these logs.",
        "  A copy of this banner is kept in FIRST-BOOT.txt in the",
        "  data volume. If you lose the code anyway, the Docker",
        "  page of the Apex docs has the one command that brings",
        "  it back — you will need access to this computer.",
        bar,
    ]
    return "\n".join(lines)


def main():
    first = os.environ.get("APEX_FIRST_BOOT") == "1"
    container = os.environ.get("APEX_CONTAINER") == "1"
    bridge = load_bridge()
    cfg = bridge.load_config()

    if container and first:
        # The posture (this node's first run only — an operator's
        # later changes to any of it are theirs and persist).
        bridge.dhd_save(cfg, {"enabled": True})
        cfg = bridge.load_config()
        block = cfg.get("mesh") if isinstance(cfg.get("mesh"), dict) else {}
        block = dict(block)
        block["enabled"] = True
        name_env = (os.environ.get("APEX_NODE_NAME") or "").strip()
        if name_env and not str(block.get("nodeName") or "").strip():
            block["nodeName"] = name_env[:80]
        ep_env = (os.environ.get("APEX_PUBLIC_ENDPOINT") or "").strip()
        if ep_env and not str(block.get("publicEndpoint") or "").strip():
            cleaned = bridge._mesh_endpoint_clean(ep_env)
            if cleaned:
                block["publicEndpoint"] = cleaned
        cfg["mesh"] = block
        bridge.save_config(cfg)
        cfg = bridge.load_config()

    if first:
        # The identity: minted here, by the bridge's own hands,
        # into the volume — the keypair and the door certificate
        # the pairing card will carry.
        bridge.mesh_ensure_identity(cfg)
        bridge.mesh_ensure_cert()
        cfg = bridge.load_config()

    if container and first:
        # The federation invitation (Build 8): the image's
        # bundled gateway card, when it carries one, becomes the
        # volume's waiting invitation. The bridge performs the
        # import itself at start (admission-only); here we only
        # place the file — first_boot invents no second import
        # path.
        bundled = os.path.join(APP, "docker", "gateway-card.png")
        waiting = os.path.join(APP, "config", "federation", "invitation.png")
        if os.path.isfile(bundled) and not os.path.exists(waiting):
            os.makedirs(os.path.dirname(waiting), exist_ok=True)
            shutil.copyfile(bundled, waiting)
            print("Apex container: a bundled gateway card rides this "
                  "image — it waits as this node's federation "
                  "invitation and is imported, admission-only, at "
                  "start.", flush=True)

    # The access code. First boot mints fresh, unconditionally:
    # whatever the shipped config carries as a placeholder is
    # not this node's code. Any later boot keeps the node's own
    # code exactly as it stands — minting again only if the code
    # is somehow absent (a volume repaired by hand), which is a
    # first-boot-class event and is announced the same way.
    minted = False
    if first:
        block = cfg.get("public_server") if isinstance(cfg.get("public_server"), dict) else {}
        block = dict(block)
        block["access_code"] = bridge.generate_access_code()
        if "enabled" not in block:
            block["enabled"] = False
        cfg["public_server"] = block
        bridge.save_config(cfg)
        code = block["access_code"]
        minted = True
    elif not bridge.public_settings(cfg)["access_code"]:
        code = bridge.ensure_access_code(cfg)
        minted = True
    else:
        code = bridge.public_settings(cfg)["access_code"]

    if minted:
        cfg = bridge.load_config()
        ms = bridge.mesh_settings(cfg)
        name = ms["nodeName"] or socket.gethostname()
        text = banner(code, name, bridge.mesh_cert_fingerprint() or "—",
                      bridge.mesh_node_id() or "")
        print(text, flush=True)
        keep = os.path.join(APP, "FIRST-BOOT.txt")
        if not os.path.exists(keep):
            with open(keep, "w", encoding="utf-8") as f:
                f.write(text + "\n")
    else:
        print("Apex container: existing node configuration found — "
              "starting with its saved identity and code "
              "(nothing re-printed).", flush=True)


if __name__ == "__main__":
    main()
