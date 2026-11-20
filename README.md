# Navi Development Library

Navi's own development library — the shipped Apex source, guides, changelog and config surface, generated from the shipped tree and version-locked to it (v0.6.9, stamps 2026-11-19.41), so she answers about her own system from the shipped truth and authors chips in the house idiom.

A **knowledge pack** for Apex — Navi's answers can draw on
these documents, and cite the pack and document they came from.
The pack format is `navi-rag/1` (see `navi-rag.json`).

## The version lock

This pack is **generated from the shipped Apex v0.6.9 tree** (build stamps 2026-11-19.41, seq 41) by `generate.py`, and from nothing else — no bench files, no operator data. It is regenerated from the shipped tree at every release; its version is the build's version. If the pack's version does not match the running build, update the pack before trusting it.

## Documents

- `documents/00-pack-guide.md`
- `documents/bridge-py-part-01.md`
- `documents/bridge-py-part-02.md`
- `documents/bridge-py-part-03.md`
- `documents/bridge-py-part-04.md`
- `documents/bridge-py-part-05.md`
- `documents/bridge-py-part-06.md`
- `documents/bridge-py-part-07.md`
- `documents/bridge-py-part-08.md`
- `documents/bridge-py-part-09.md`
- `documents/bridge-py-part-10.md`
- `documents/bridge-py-part-11.md`
- `documents/bridge-py-part-12.md`
- `documents/bridge-py-part-13.md`
- `documents/app-index-html-part-01.md`
- `documents/app-index-html-part-02.md`
- `documents/app-index-html-part-03.md`
- `documents/app-index-html-part-04.md`
- `documents/app-index-html-part-05.md`
- `documents/app-index-html-part-06.md`
- `documents/app-index-html-part-07.md`
- `documents/app-index-html-part-08.md`
- `documents/app-index-html-part-09.md`
- `documents/changelog-part-01.md`
- `documents/changelog-part-02.md`
- `documents/apex-readme.md`
- `documents/install-sh.md`
- `documents/apply-helper-and-bootstrap-py.md`
- `documents/intel-py.md`
- `documents/guardian-py.md`
- `documents/wire-py.md`
- `documents/satellite-modules-py.md`
- `documents/operator-guides.md`
- `documents/chip-format.md`
- `documents/shell-sources.md`
- `documents/config-surface.md`
- `documents/startup-and-services.md`

## Install it in Navi

1. Open Navi's **RAG** tab, then **Packs**.
2. **Search GitHub** — every public repo tagged `navi-rag` is listed.
3. **Install** this pack. Navi downloads the documents at the latest
   release and builds the pack's index on your machine from them.

Only **active** packs feed her answers; toggle the pack off in the
same tab whenever you like.

## Layout

- `README.md` — this page. Navi's installer skips a root README, so
  it describes the pack without ever being indexed as a document.
- `navi-rag.json` — the pack manifest (name, version, document list).
- `index.json` — the pack's built index, shipped as a reference copy;
  installing Navi rebuilds it locally from the documents.
- `documents/` — the pack's documents (`.md` / `.txt`).
- `generate.py` — the generator that built this pack from the shipped
  tree (stdlib only). It is not a document and is never indexed.

## Publishing an update

Regenerate with `generate.py` against the new release's shipped tree,
bump `version` in `navi-rag.json` (generate.py does this from the
tree's own stamps), push, and publish a new release — the version tag
is created with that publish. Keep the `navi-rag` topic on the
repo — that is how Navi finds it.
