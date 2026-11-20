# The Navi Development Library — a guide to this pack

This document is GENERATED (by generate.py, from the shipped tree) — it is the pack's own map of itself. Everything else in the pack is a shipped file reproduced verbatim.

## What this pack is

Navi's development library: the source, guides, changelog and config surface of the Apex build she is running, as retrievable knowledge. It exists so she can answer questions about her own system from the shipped truth — and author chips in the house idiom, from the code that actually ships.

## The version lock

- Generated from: the shipped Apex **v0.6.9** tree (VERSION file: `0.6.9 edge`).
- Build stamps: **2026-11-19.41** / seq **41** — the same stamps the running bridge reports about itself.
- Regeneration rule: this pack is regenerated from the shipped tree at every release and its version moves with the build. A library that does not match the running build teaches dead APIs, so an out-of-date copy of this pack should be updated, not trusted.

## Quick facts — canonical values, straight from the tree

Which port does the bridge serve on? The bridge serves on port 8675 (BRIDGE_PORT in bridge.py). The llamafile model server endpoint is http://localhost:8080/v1.

Where does Apex install on a machine? Apex installs into ~/.local/share/navi-apex — the install home (DEFAULT_HOME in install.sh). A legacy /opt/navi-apex install is migrated into that home.

Which ports does the Mesh door listen on? The Mesh door's canonical port is 8765; the shipped default listen ports add 8443, 2475 — one door, one certificate, one fingerprint on every port (MESH_DEFAULT_EXTRA_PORTS in bridge.py).

How does Navi discover knowledge packs and chips? Knowledge packs are GitHub repos carrying the navi-rag topic (the catalog search is `topic:navi-rag`); chips are GitHub repos carrying the navi-chips topic.

How is a chip stored? A chip's entire codebase rides inside its PNG image in a tEXt chunk with the keyword "navi-chip" (JSON: name, description, code, output). Share the image and you share the chip.

How large can a knowledge pack be? A knowledge pack holds at most 40 documents, and a document body is capped at 200000 characters at install. The pack index is feature-hashed term vectors of 256 dimensions; retrieval keeps chunks scoring at least 0.18 against the question.

Which build is this library locked to? This pack is version-locked to Apex v0.6.9, build stamps 2026-11-19.41, seq 41.

## How the pack is organised

Large sources are split into numbered parts at symbol boundaries (never mid-function where a boundary exists); each part's header names its source file and line range in the shipped tree, so a retrieved passage can be traced back to the exact lines. Documents:

- `documents/bridge-py-part-01.md` — bridge.py, part 01 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-02.md` — bridge.py, part 02 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-03.md` — bridge.py, part 03 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-04.md` — bridge.py, part 04 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-05.md` — bridge.py, part 05 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-06.md` — bridge.py, part 06 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-07.md` — bridge.py, part 07 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-08.md` — bridge.py, part 08 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-09.md` — bridge.py, part 09 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-10.md` — bridge.py, part 10 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-11.md` — bridge.py, part 11 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-12.md` — bridge.py, part 12 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/bridge-py-part-13.md` — bridge.py, part 13 — the bridge: chat, Mesh, cases, sync, Seraph, RAG, the updater and everything else.
- `documents/app-index-html-part-01.md` — app/index.html, part 01 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-02.md` — app/index.html, part 02 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-03.md` — app/index.html, part 03 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-04.md` — app/index.html, part 04 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-05.md` — app/index.html, part 05 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-06.md` — app/index.html, part 06 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-07.md` — app/index.html, part 07 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-08.md` — app/index.html, part 08 — the single-file web UI: every screen she wears.
- `documents/app-index-html-part-09.md` — app/index.html, part 09 — the single-file web UI: every screen she wears.
- `documents/changelog-part-01.md` — CHANGELOG.md, part 01 — her whole shipped history, version by version.
- `documents/changelog-part-02.md` — CHANGELOG.md, part 02 — her whole shipped history, version by version.
- `documents/apex-readme.md` — The Apex README — what Apex is, in its own words.
- `documents/install-sh.md` — install.sh — the installer: install homes, migration, markers, service registration.
- `documents/apply-helper-and-bootstrap-py.md` — The update machinery: apply-helper.py and apply-bootstrap.py.
- `documents/intel-py.md` — intel.py — the Wire's intel collection and watchlist machinery.
- `documents/guardian-py.md` — guardian.py — the Guardian that watches the box itself.
- `documents/wire-py.md` — wire.py — the Wire's feed machinery.
- `documents/satellite-modules-py.md` — rowfeed.py, bounties.py and ctf.py — the smaller shipped modules.
- `documents/operator-guides.md` — The shipped operator guides: docs/MESH.md, docs/DHD.md, docs/FEDERATION.md.
- `documents/chip-format.md` — How chips work — the cartridge format and the Chips screen (chips/README.md).
- `documents/shell-sources.md` — The desktop shell: shell/src/main.rs, its README, Cargo.toml and tauri.conf.json.
- `documents/config-surface.md` — The config surface: default config, Soul and Memory seeds, sign-in client, VERSION, MANIFEST.txt.
- `documents/startup-and-services.md` — start.sh, run-llamafile.sh, the systemd units and the Docker entrypoint.

## What is NOT in here

Only the shipped tree is an input. Bench suites, campaign files, build packets, operator data, memories and case material are not part of the shipped tree and are never in this pack.
