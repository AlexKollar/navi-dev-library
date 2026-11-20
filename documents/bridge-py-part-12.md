# bridge.py — part 12 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 44873–46081 of bridge.py.

                            self._send(200, {"ok": False, "error": path + " has no extractable text — a scanned or image-only PDF can't be indexed."})
                            return
                    else:
                        try:
                            with open(path) as f:
                                body = f.read()
                        except Exception as e:
                            self._send(200, {"ok": False, "error": "Could not read " + path + ": " + str(e)})
                            return
                    documents.append({
                        "title": os.path.relpath(path, root),
                        "body": body[:RAG_MAX_DOC_CHARS],
                    })
            for raw in data.get("documents") or []:
                if not isinstance(raw, dict):
                    continue
                title = str(raw.get("title") or "document")[:200]
                if raw.get("pdf_base64"):
                    try:
                        body = pdf_extract_text(base64.b64decode(str(raw.get("pdf_base64"))))
                    except Exception:
                        body = ""
                    if not body:
                        self._send(200, {"ok": False, "error": title + " has no extractable text — a scanned or image-only PDF can't be indexed."})
                        return
                    documents.append({
                        "title": title,
                        "body": body[:RAG_MAX_DOC_CHARS],
                    })
                elif str(raw.get("body") or "").strip():
                    documents.append({
                        "title": title,
                        "body": str(raw.get("body"))[:RAG_MAX_DOC_CHARS],
                    })
            documents = documents[:RAG_MAX_DOCS]
            icon_rows, icon_err = _payload_icon(data)
            if icon_err:
                self._send(200, {"ok": False, "error": icon_err})
            elif not name:
                self._send(200, {"ok": False, "error": "A pack needs a name."})
            elif not documents:
                self._send(200, {"ok": False, "error": "A pack needs at least one document — pick files in the Studio or name a folder on disk."})
            else:
                rows = load_rag_installs()
                slug = chip_slug(name)
                taken = {
                    r.get("local_dir") for r in rows
                    if r.get("full_name") != "local/" + slug
                }
                base = slug
                n = 2
                while slug in taken:
                    slug = base + "-" + str(n)
                    n += 1
                write_rag_pack(slug, name, description, documents,
                               icon=icon_rows or None)
                rows = [r for r in rows if r.get("full_name") != "local/" + slug]
                rows.append({
                    "full_name": "local/" + slug,
                    "name": name,
                    "html_url": "",
                    "installed_tag": None,
                    "installed_ref": "forged in RAG Studio",
                    "installed_pushed_at": None,
                    "installed_at": _now_iso(),
                    "local_dir": slug,
                    "active": True,
                    "doc_count": len(documents),
                })
                save_rag_installs(rows)
                self._send(200, {"ok": True, "error": None, "slug": slug})
        elif self.path == "/api/rag/update-pack":
            # RAG Studio saving an edit to a pack already in the
            # folder: the new name, description, and document set
            # (documents added or removed in the Studio; PDFs travel
            # as base64 and contribute their extracted text, exactly
            # like the forge) replace the pack's contents in its own
            # folder, and the index rebuilds from the new set. The
            # pack's active flag survives. A GitHub-installed pack is
            # marked locally_edited — the divergence from its
            # published source is stated, never silent, and a catalog
            # update refuses rather than overwriting the edits.
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            name = str(data.get("name") or "").strip()[:120]
            description = str(data.get("description") or "")[:500]
            documents = []
            for raw in data.get("documents") or []:
                if not isinstance(raw, dict):
                    continue
                title = str(raw.get("title") or "document")[:200]
                if raw.get("pdf_base64"):
                    try:
                        body = pdf_extract_text(base64.b64decode(str(raw.get("pdf_base64"))))
                    except Exception:
                        body = ""
                    if not body:
                        self._send(200, {"ok": False, "error": title + " has no extractable text — a scanned or image-only PDF can't be indexed."})
                        return
                    documents.append({"title": title, "body": body[:RAG_MAX_DOC_CHARS]})
                elif str(raw.get("body") or "").strip():
                    documents.append({"title": title, "body": str(raw.get("body"))[:RAG_MAX_DOC_CHARS]})
            documents = documents[:RAG_MAX_DOCS]
            icon_rows, icon_err = _payload_icon(data)
            if icon_err:
                self._send(200, {"ok": False, "error": icon_err})
                return
            if not slug or _rag_pack_index(slug) is None:
                self._send(200, {"ok": False, "error": "That pack is not in the folder."})
            elif not name:
                self._send(200, {"ok": False, "error": "A pack needs a name."})
            elif not documents:
                self._send(200, {"ok": False, "error": "A pack needs at least one document — add one in the Studio before saving."})
            else:
                write_rag_pack(slug, name, description, documents,
                               icon=icon_rows or None)
                rows = load_rag_installs()
                for r in rows:
                    if r.get("local_dir") == slug:
                        r["name"] = name
                        r["doc_count"] = len(documents)
                        if not str(r.get("full_name") or "").startswith("local/"):
                            r["locally_edited"] = True
                save_rag_installs(rows)
                self._send(200, {"ok": True, "error": None, "slug": slug})
        elif self.path == "/api/rag/export":
            # Export for GitHub: build the pack's repo ZIP in
            # rag-exports/ — README page, manifest, the built
            # index.json (reference copy; installs rebuild it),
            # documents/, .gitignore.
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                zip_name, repo_files = export_rag_pack(slug, version)
                self._send(200, {"ok": True, "error": None, "file": zip_name, "files": repo_files})
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e), "file": None, "files": []})
        elif self.path == "/api/rag/export-png":
            # Export as a knowledge cartridge (issue #93): the pack
            # as ONE picture in rag-exports/, beside the ZIP export.
            # Past the size ceiling the refusal is honest — the ZIP
            # stays the way to carry a big pack — and nothing is
            # truncated to fit.
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            payload = rag_cartridge_payload(slug)
            if payload is None:
                self._send(200, {"ok": False, "error": "That pack is not in rag-packs/.", "file": None})
                return
            payload["version"] = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            png = build_rag_cartridge_png(payload)
            if len(png) > RAG_CARTRIDGE_MAX_BYTES:
                self._send(200, {"ok": False, "error": "This pack is too big for a picture — its cartridge would run " + str(len(png)) + " bytes against the " + str(RAG_CARTRIDGE_MAX_BYTES) + "-byte ceiling. Use the ZIP export for this one; nothing was written.", "file": None})
                return
            os.makedirs(RAG_EXPORTS_DIR, exist_ok=True)
            fname = slug + "-cartridge.png"
            with open(os.path.join(RAG_EXPORTS_DIR, fname), "wb") as f:
                f.write(png)
            self._send(200, {"ok": True, "error": None, "file": fname, "size": len(png)})
        elif self.path == "/api/rag/install-cartridge":
            # Install a pack from its picture alone (issue #93):
            # the dashboard sends the picture's bytes base64'd;
            # the payload validates and installs through the one
            # cartridge path, index built locally at install.
            data = self._body()
            try:
                raw = base64.b64decode(str(data.get("png") or ""), validate=True)
            except Exception:
                raw = None
            if not raw:
                self._send(200, {"ok": False, "error": "That upload did not arrive as a picture — try the upload again."})
                return
            if len(raw) > RAG_CARTRIDGE_MAX_BYTES:
                self._send(200, {"ok": False, "error": "That file is larger than a knowledge cartridge may be (" + str(RAG_CARTRIDGE_MAX_BYTES) + " bytes) — nothing was installed."})
                return
            payload, perr = parse_rag_cartridge_png(raw)
            if payload is None:
                self._send(200, {"ok": False, "error": perr})
                return
            slug, ierr = install_rag_cartridge(payload)
            if slug is None:
                self._send(200, {"ok": False, "error": ierr})
                return
            self._send(200, {"ok": True, "error": None, "slug": slug})
        elif self.path == "/api/rag/publish":
            # Publish to GitHub: upload the export's exact file set
            # to a repo on the signed-in account (created when
            # missing, navi-rag topic applied), and — since
            # 0.1.9e — the tag and the PUBLISHED release too,
            # with the knowledge cartridge attached. Publish
            # does everything; there is no manual release step.
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                files = rag_repo_files(slug, version)
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e)})
            else:
                index = _rag_pack_index(slug) or {}
                repo_name = str(data.get("repo_name") or "").strip() or (index.get("name") or slug)
                # The knowledge cartridge rides along: at the repo
                # root as the pack's picture, and attached to the
                # release. A pack too big for a picture publishes
                # without one, and says so in the result.
                asset_skip = None
                cart_name = slug + "-cartridge.png"
                cart_png = None
                try:
                    cart_png = build_rag_cartridge_png(rag_cartridge_payload(slug))
                    if len(cart_png) > RAG_CARTRIDGE_MAX_BYTES:
                        asset_skip = ("The pack is too big for a picture ("
                                      + str(len(cart_png)) + " bytes against the "
                                      + str(RAG_CARTRIDGE_MAX_BYTES) + "-byte ceiling), so the release carries no cartridge — the files and documents are all there. The ZIP export carries the pack whole.")
                        cart_png = None
                except Exception as e:
                    cart_png = None
                    asset_skip = "The knowledge cartridge could not be built (" + str(e) + "), so the release carries no cartridge — the files and documents are all there."
                if cart_png:
                    files = files + [(cart_name, cart_png)]
                _rreadme = next((d for a, d in files if a == "README.md"), "")
                release = {
                    "version": version,
                    "title": (index.get("name") or slug) + " v" + version.lstrip("v"),
                    "notes": _rreadme if isinstance(_rreadme, str) else "",
                    "assetName": cart_name if cart_png else "",
                    "assetData": cart_png,
                }
                result = github_publish_files(repo_name, files, "navi-rag", index.get("description") or "", release)
                if asset_skip and result.get("ok"):
                    result["assetSkipped"] = asset_skip
                self._send(200, result)
        elif self.path == "/api/rag/readme":
            # The effective README for the editor: the operator's
            # saved edit when there is one, else the generated page.
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                files = rag_repo_files(slug, version)
                readme = next((d for a, d in files if a == "README.md"), "")
                self._send(200, {"ok": True, "error": None, "readme": readme if isinstance(readme, str) else "", "edited": rag_readme_override(slug) is not None})
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e), "readme": "", "edited": False})
        elif self.path == "/api/rag/readme-save":
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            text = str(data.get("readme") or "")
            if _rag_pack_index(slug) is None:
                self._send(200, {"ok": False, "error": "That pack is not in rag-packs/."})
            elif len(text) > 200000:
                self._send(200, {"ok": False, "error": "That README is too long to keep (200,000 characters max)."})
            else:
                with open(os.path.join(RAG_PACKS_DIR, slug, ".readme-override"), "w") as f:
                    f.write(text)
                self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/rag/readme-restore":
            data = self._body()
            slug = os.path.basename(str(data.get("slug") or ""))
            try:
                os.remove(os.path.join(RAG_PACKS_DIR, slug, ".readme-override"))
            except Exception:
                pass
            self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/chips/export":
            # Export for GitHub: build a local chip's repo ZIP
            # in chips-exports/ — README page, navi-chip.json, the
            # chip's files at the root, the chip, .gitignore.
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                zip_name, repo_files = export_chip(fname, version)
                self._send(200, {"ok": True, "error": None, "file": zip_name, "files": repo_files})
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e), "file": None, "files": []})
        elif self.path == "/api/chips/publish":
            # Publish to GitHub: the chip counterpart of the pack
            # publish — same loop, navi-chips topic.
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                chip = load_local_chip(fname)
                files = build_chip_repo_files(chip, version)
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e)})
            else:
                repo_name = str(data.get("repo_name") or "").strip() or chip.get("name") or ""
                _png = next(((a, d) for a, d in files if a.endswith(".png") and isinstance(d, (bytes, bytearray))), (None, None))
                _readme = next((d for a, d in files if a == "README.md"), "")
                release = {
                    "version": version,
                    "title": (chip.get("name") or "Chip") + " v" + version.lstrip("v"),
                    "notes": _readme if isinstance(_readme, str) else "",
                    "assetName": _png[0] or "",
                    "assetData": _png[1],
                }
                result = github_publish_files(repo_name, files, "navi-chips", chip.get("description") or "", release)
                self._send(200, result)
        elif self.path == "/api/chips/readme":
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            version = str(data.get("version") or "0.1.0").strip()[:40] or "0.1.0"
            try:
                chip = load_local_chip(fname)
                files = build_chip_repo_files(chip, version)
                readme = next((d for a, d in files if a == "README.md"), "")
                self._send(200, {"ok": True, "error": None, "readme": readme if isinstance(readme, str) else "", "edited": bool(chip.get("readme"))})
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e), "readme": "", "edited": False})
        elif self.path == "/api/chips/readme-save":
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            text = str(data.get("readme") or "")
            if len(text) > 200000:
                self._send(200, {"ok": False, "error": "That README is too long to keep (200,000 characters max)."})
            else:
                try:
                    save_local_chip_readme(fname, text)
                    self._send(200, {"ok": True, "error": None})
                except Exception as e:
                    self._send(200, {"ok": False, "error": str(e)})
        elif self.path == "/api/chips/readme-restore":
            data = self._body()
            fname = os.path.basename(str(data.get("file") or ""))
            try:
                save_local_chip_readme(fname, None)
                self._send(200, {"ok": True, "error": None})
            except Exception as e:
                self._send(200, {"ok": False, "error": str(e)})
        elif self.path == "/api/deck/install":
            # Drop-to-install on ChipDeck (0.1.9e, #94): one door
            # for every picture — the payload keyword inside the
            # PNG says what it is, and each kind installs through
            # its own kind's one path (chip picture install, the
            # knowledge-cartridge path, the Mesh card pairing).
            data = self._body()
            try:
                raw = base64.b64decode(str(data.get("png") or ""), validate=True)
            except Exception:
                raw = None
            if not raw:
                self._send(200, {"ok": False, "error": "That upload did not arrive as a picture — try the upload again."})
                return
            keys = _png_text_keywords(raw)
            if "navi-mesh-card" in keys:
                peer, perr = parse_mesh_card_png(raw)
                if peer is None:
                    self._send(200, {"ok": False, "error": perr})
                    return
                err = mesh_add_peer(cfg, peer)
                if err:
                    self._send(200, {"ok": False, "error": err})
                    return
                # The card's picture joins the binder's card folder
                # (the backup's naming: peer-<node>.png) — pairing
                # through the Deck keeps the picture it paired from,
                # exactly as a backup restore lands it.
                try:
                    os.makedirs(MESH_CARDS_DIR, exist_ok=True)
                    cname = "peer-" + str(peer.get("nodeId") or "card")[:12] + ".png"
                    with open(os.path.join(MESH_CARDS_DIR, cname), "wb") as f:
                        f.write(raw)
                except Exception:
                    pass
                mesh_probe_peers_soon()
                self._send(200, {"ok": True, "error": None, "kind": "mesh", "name": peer.get("name") or ""})
                return
            if "navi-rag" in keys:
                payload, perr = parse_rag_cartridge_png(raw)
                if payload is None:
                    self._send(200, {"ok": False, "error": perr})
                    return
                slug, ierr = install_rag_cartridge(payload)
                if slug is None:
                    self._send(200, {"ok": False, "error": ierr})
                    return
                self._send(200, {"ok": True, "error": None, "kind": "rag", "name": payload.get("name") or slug})
                return
            if "navi-chip" in keys:
                payload = parse_cartridge_png(raw)
                if payload is None:
                    self._send(200, {"ok": False, "error": "That picture's chip data could not be read — it may have been recompressed on its way here, which strips the data out. Send the original file inside a .zip instead."})
                    return
                slug = chip_slug(payload["name"])
                filename = slug + ".png"
                if os.path.isfile(os.path.join(CHIPS_DIR, filename)):
                    self._send(200, {"ok": False, "error": payload["name"] + " is already on this machine — its picture is in the chips folder."})
                    return
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, filename), "wb") as f:
                    f.write(raw)
                rows = [r for r in load_chip_installs() if r.get("full_name") != "local/" + slug]
                rows.append({
                    "full_name": "local/" + slug,
                    "name": payload["name"],
                    "html_url": "",
                    "installed_tag": None,
                    "installed_ref": "chip " + filename + " (from its picture)",
                    "installed_pushed_at": None,
                    "installed_at": _now_iso(),
                    "local_file": filename,
                    "requirements": list(payload.get("requirements") or []),
                })
                save_chip_installs(rows)
                self._send(200, {"ok": True, "error": None, "kind": "chip", "name": payload["name"], "version": str(payload.get("version") or "")})
                return
            self._send(200, {"ok": False, "error": "That picture carries nothing the binder can slot in — no chip, knowledge cartridge, or Mesh card data inside it. A picture that was recompressed on its way here loses its data while looking the same; send the original inside a .zip."})
        elif self.path == "/api/chips/install":
            # Installing a chip means the chip itself lands here:
            # its source is downloaded from the repo at the ref
            # being installed and forged into a chip in chips/,
            # and the install record points at that chip. A
            # download that fails installs nothing — no record for a
            # chip that is not actually on this machine.
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_chip_installs()
            if any(r.get("full_name") == full for r in rows):
                self._send(200, {"ok": False, "error": "That chip is already installed."})
            else:
                try:
                    repo = fetch_repo_chip(full)
                    payload, source_file = fetch_chip_payload(repo)
                except Exception as e:
                    self._send(200, {"ok": False, "error": "The chip could not be downloaded from GitHub, so nothing was installed: " + str(e)})
                    return
                filename = chip_cartridge_filename(repo, rows)
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, filename), "wb") as f:
                    f.write(build_cartridge_png(payload))
                record = chip_install_record(repo)
                record["local_file"] = filename
                record["source_file"] = source_file
                record["requirements"] = list(payload.get("requirements") or [])
                rows.append(record)
                save_chip_installs(rows)
                # Build .86 (issue #79): the install verifies
                # itself — re-run the folder scan and answer ok
                # only if the chip actually appears in it. A chip
                # that landed on disk but does not list is named
                # as such, with the reason, and its record is
                # taken back so Installed never claims it.
                landed = next((c for c in local_chips() if c.get("file") == filename), None)
                if landed is None or landed.get("unreadable"):
                    rows = [r for r in rows if r is not record]
                    save_chip_installs(rows)
                    if landed is None and not os.path.isfile(os.path.join(CHIPS_DIR, filename)):
                        why = "the chip file is missing from the chips folder"
                    else:
                        why = "its cartridge could not be read back (unparseable cartridge)"
                    self._send(200, {"ok": False, "error": "The chip landed on disk as " + filename + " but does not list: " + why + ". Nothing was kept as installed."})
                    return
                self._send(200, {"ok": True, "error": None, "version": str(payload.get("version") or "")})
        elif self.path == "/api/chips/update":
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_chip_installs()
            inst = next((r for r in rows if r.get("full_name") == full), None)
            if inst is None:
                self._send(200, {"ok": False, "updated": False, "error": "That chip is not installed."})
            elif inst.get("locally_edited"):
                self._send(200, {"ok": False, "updated": False, "error": "This chip has local edits (edited in Chip Studio), so the update was not applied — it would overwrite them. Export the edited chip for GitHub to publish your version, or delete the install and reinstall to return to the published source."})
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
                filename = inst.get("local_file")
                have_cartridge = bool(filename) and os.path.isfile(os.path.join(CHIPS_DIR, filename))
                if not newer and have_cartridge:
                    self._send(200, {"ok": True, "updated": False, "error": None})
                    return
                # Newer upstream — or the record predates chips
                # landing on disk and the chip itself is missing:
                # (re)download the source at the latest ref and
                # re-forge the chip, so what runs here is what
                # the repo now ships. A failed download leaves the
                # installed copy exactly as it was.
                try:
                    payload, source_file = fetch_chip_payload(repo)
                except Exception as e:
                    self._send(200, {"ok": False, "updated": False, "error": "The chip could not be downloaded from GitHub, so the installed copy was left as it was: " + str(e)})
                    return
                if not filename:
                    filename = chip_cartridge_filename(repo, rows)
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, filename), "wb") as f:
                    f.write(build_cartridge_png(payload))
                fresh = chip_install_record(repo)
                fresh["installed_at"] = inst.get("installed_at")
                fresh["local_file"] = filename
                fresh["source_file"] = source_file
                fresh["requirements"] = list(payload.get("requirements") or [])
                rows[rows.index(inst)] = fresh
                save_chip_installs(rows)
                self._send(200, {"ok": True, "updated": True, "error": None})
        elif self.path == "/api/chips/delete":
            data = self._body()
            full = str(data.get("full_name") or "")
            rows = load_chip_installs()
            target = next((r for r in rows if r.get("full_name") == full), None)
            rows = [r for r in rows if r.get("full_name") != full]
            save_chip_installs(rows)
            # The chip's chip goes with its record — true for
            # Studio-forged chips and GitHub-downloaded ones alike:
            # deleting an install takes the chip off the shelf.
            if target and target.get("local_file"):
                try:
                    os.remove(os.path.join(CHIPS_DIR, target["local_file"]))
                except Exception:
                    pass
            elif full.startswith("local/"):
                # 0.1.9e audit fix: a chip dropped into chips/ by
                # hand has NO install record, so the record-driven
                # delete above removed nothing while the UI said
                # "Deleted" — the cartridge stayed on the shelf.
                # A local/ delete names the file by its stem; take
                # the picture with the (absent) record.
                _stem = re.sub(r"[^a-z0-9-]+", "-", full[len("local/"):].lower()).strip("-")
                if _stem:
                    try:
                        os.remove(os.path.join(CHIPS_DIR, _stem + ".png"))
                    except Exception:
                        pass
            self._send(200, {"ok": True, "error": None})
        elif self.path == "/api/chips/create":
            # Chip Studio: forge a new chip into a .png chip —
            # code embedded in the image — save it into chips/, and
            # register it as a local install.
            data = self._body()
            name = str(data.get("name") or "").strip()[:120]
            code = str(data.get("code") or "")
            if not name or not code.strip():
                self._send(200, {"ok": False, "error": "A chip needs a name and code.", "filename": None})
            else:
                icon_rows, icon_err = _payload_icon(data)
                if icon_err:
                    self._send(200, {"ok": False, "error": icon_err, "filename": None})
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
                slug = re.sub(r"[^a-z0-9]+", "-", name.lower()).strip("-") or "chip"
                filename = slug + ".png"
                os.makedirs(CHIPS_DIR, exist_ok=True)
                with open(os.path.join(CHIPS_DIR, filename), "wb") as f:
                    f.write(build_cartridge_png(payload))
                rows = [r for r in load_chip_installs() if r.get("full_name") != "local/" + slug]
                rows.append({
                    "full_name": "local/" + slug,
                    "name": name,
                    "html_url": "",
                    "installed_tag": None,
                    "installed_ref": "chip " + filename,
                    "installed_pushed_at": None,
                    "installed_at": _now_iso(),
                    "local_file": filename,
                })
                save_chip_installs(rows)
                self._send(200, {"ok": True, "error": None, "filename": filename})
        elif self.path == "/api/github/client-id":
            # The OAuth App client id is a config value (it must be
            # registered under the project owner's GitHub account, so
            # none ships). It is not a secret — Device Flow has none.
            data = self._body()
            save_github_settings({"client_id": str(data.get("client_id") or "").strip()[:80]})
            self._send(200, github_status_payload(load_config()))
        elif self.path == "/api/github/settings":
            data = self._body()
            patch = {}
            repo = str(data.get("backup_repo") or "").strip()
            if repo:
                if not re.match(r"^[A-Za-z0-9._-]+$", repo) or repo.startswith(".") or repo.startswith("-"):
                    self._send(200, {"ok": False, "error": "Repository names are letters, digits, dots, dashes, and underscores."})
                    return
                patch["backup_repo"] = repo[:100]
            if isinstance(data.get("backup_private"), bool):
                patch["backup_private"] = data["backup_private"]
            if isinstance(data.get("backup_items"), list):
                chosen = [i for i in data["backup_items"] if i in GITHUB_BACKUP_ITEMS]
                patch["backup_items"] = [i for i in GITHUB_BACKUP_ITEMS if i in GITHUB_CORE_BACKUP_ITEMS or i in chosen]
            if data.get("backup_interval_days") in (1, 3, 5, 7):
                patch["backup_interval_days"] = data["backup_interval_days"]
            save_github_settings(patch)
            self._send(200, github_status_payload(load_config()))
        elif self.path == "/api/github/signin/start":
            self._send(200, github_device_start(cfg))
        elif self.path == "/api/github/signin/poll":
            self._send(200, github_device_poll(cfg))
        elif self.path == "/api/github/signout":
            # Signing out deletes the only copy of the token. Navi
            # herself is untouched — she never needed it.
            _github_device.clear()
            try:
                os.remove(GITHUB_TOKEN_PATH)
            except Exception:
                pass
            self._send(200, github_status_payload(load_config()))
        elif self.path == "/api/chats/sync":
            # Write-through (#139): the writing window's whole set,
            # authoritative for the store it owns.
            data = self._body()
            rows = data.get("chats") if isinstance(data.get("chats"), list) else []
            ups = data.get("uploads") if isinstance(data.get("uploads"), list) else None
            self._send(200, chats_replace_all(rows, ups))
        elif self.path == "/api/chats/put":
            data = self._body()
            saved = chat_write(data.get("chat") if isinstance(data.get("chat"), dict) else None)
            if saved is None:
                self._send(400, {"ok": False, "error": "That chat record was refused — it needs a usable id and a messages list."})
            else:
                self._send(200, {"ok": True, "chat": saved})
        elif self.path == "/api/chats/delete":
            data = self._body()
            if str(data.get("id") or "").strip() == MAIN_CHAT_ID:
                # v0.3.0e Build 1 (#200): refused BY NAME — the
                # main chat is emptied, never deleted.
                self._send(200, {"ok": False, "error":
                                 "The main chat can't be deleted — "
                                 "empty it with /clear or /cls."})
            else:
                self._send(200, {"ok": chat_delete(data.get("id"))})
        elif self.path == "/api/cases/create":
            data = self._body()
            detail, err = case_create(data.get("name"), data.get("client"), data.get("scope"),
                                      data.get("fields") if isinstance(data.get("fields"), dict) else None)
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                # Build 15 (#161): a case is born with its chat.
                chat = case_chat_ensure(detail["slug"])
                detail["chatId"] = chat["id"] if chat else None
                self._send(200, {"ok": True, "case": detail})
        elif self.path == "/api/cases/file":
            data = self._body()
            filed, err = case_file_entry(str(data.get("case") or ""), str(data.get("kind") or "finding"),
                                         data.get("title"), data.get("body"),
                                         str(data.get("severity") or "info"), data.get("internal") is True)
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "filed": filed})
        elif self.path == "/api/cases/delete-finding":
            # v0.5.9e (#280): the case door for element/finding
            # deletion. A finding carrying a dossier element
            # routes to the global delete (every home); any
            # other finding is removed from this case alone.
            # Either way the case's custody record gains the
            # line and the response carries the fresh detail.
            data = self._body()
            with DOSSIER_LOCK:
                _res, _err = case_delete_finding(
                    str(data.get("case") or ""),
                    str(data.get("file") or ""),
                    str(data.get("reason") or ""))
            if _err:
                self._send(200, {"ok": False, "error": _err})
            else:
                self._send(200, {"ok": True, "deleted": _res,
                                 "case": case_detail_payload(
                                     str(data.get("case") or ""))})
        elif self.path == "/api/cases/file-reply":
            # v0.5.2e (#261, leg 1): the deterministic filing
            # door — one of Navi's replies in a case chat, filed
            # into its bound case by the operator's tap, with
            # provenance, the grounding guard, and the receipt
            # appended into the chat by the bridge itself.
            data = self._body()
            filing, err = case_file_reply(data.get("chatId"), data.get("text"),
                                          data.get("part"))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "filed": filing})
        elif self.path == "/api/cases/start-from-reply":
            # v0.5.14e (#289): the "Start a case from this"
            # door — the operator's press on one of Navi's
            # replies in a chat that is not a case room. The
            # bridge composes the case from the reply exactly
            # as printed and receipts it into that chat; a
            # refusal is named and creates nothing.
            data = self._body()
            started, err = case_start_from_reply(data.get("chatId"), data.get("text"),
                                                 data.get("part"))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "started": started})
        elif self.path == "/api/cases/evidence":
            data = self._body()
            try:
                raw = base64.b64decode(str(data.get("dataBase64") or ""))
            except Exception:
                raw = b""
            saved, err = case_add_evidence(str(data.get("case") or ""), data.get("name"), raw,
                                           str(data.get("description") or ""),
                                           str(data.get("acquired") or ""),
                                           str(data.get("custody") or ""))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "evidence": saved})
        elif self.path == "/api/cases/update":
            # Build 2 (#199): the manifest edit door — summary,
            # type, priority, tags, sensitive, owner, related,
            # close-out. The sensitive toggle rides it too.
            data = self._body()
            detail, err = case_update(str(data.get("case") or ""),
                                      data.get("fields") if isinstance(data.get("fields"), dict) else {})
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "case": detail})
        elif self.path == "/api/cases/register":
            # Build 2 (#199): the operator files a register
            # entry (entity / observable / task) from the view.
            # v0.4.5e (#241): with "unverified" set, the entry
            # rides the lead door instead — Navi-swiped material
            # (the New case form's checked entries) lands as a
            # lead in her name, never as the operator's record.
            data = self._body()
            if data.get("unverified") is True:
                detail, err = case_register_add_lead(
                    str(data.get("case") or ""), str(data.get("register") or ""),
                    data.get("heading"),
                    data.get("fields") if isinstance(data.get("fields"), dict) else {},
                    data.get("body"))
            else:
                detail, err = case_register_add(
                    str(data.get("case") or ""), str(data.get("register") or ""),
                    data.get("heading"),
                    data.get("fields") if isinstance(data.get("fields"), dict) else {},
                    data.get("body"))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "case": detail})
        elif self.path == "/api/cases/review":
            # Build 2 (#199): one-tap review — promote or
            # strike an unverified entry (findings, entities,
            # observables), or check a task off / reopen it.
            data = self._body()
            detail, err = case_review(str(data.get("case") or ""),
                                      str(data.get("register") or ""),
                                      str(data.get("key") or ""),
                                      str(data.get("action") or ""))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "case": detail})
        elif self.path == "/api/cases/report":
            data = self._body()
            text, withheld, err = case_report(str(data.get("case") or ""))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "report": text, "withheld": withheld})
        elif self.path == "/api/cases/open":
            data = self._body()
            slug = str(data.get("case") or "")
            cfg2 = load_config()
            block = cfg2.get("casefiles") if isinstance(cfg2.get("casefiles"), dict) else {}
            chat_id = None
            if slug and _case_manifest(slug) is not None:
                block["open"] = slug
                # Build 15 (#161): opening a case ensures its
                # chat — created on first open, returned to after.
                chat = case_chat_ensure(slug)
                chat_id = chat["id"] if chat else None
            else:
                block.pop("open", None)
                slug = ""
            cfg2["casefiles"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "open": slug, "chatId": chat_id})
        elif self.path == "/api/cases/chat":
            # Build 15 (#161): the case chat's door — ensure the
            # tied chat and hand back its id, for the Case Files
            # view's jump into the room.
            data = self._body()
            slug = str(data.get("case") or "")
            chat = case_chat_ensure(slug) if slug else None
            if chat is None:
                self._send(200, {"ok": False, "error": "No such case."})
            else:
                self._send(200, {"ok": True, "chatId": chat["id"], "chat": chat})
        elif self.path == "/api/cases/shelf":
            # v0.5.10e (#283): the reference shelf's door — the
            # Wire page's per-card Pull lands here (add), the
            # case chat's shelf removes here (remove), and list
            # hands back the shelf's metadata. The shelf is the
            # case CHAT's state: nothing here touches the case
            # folder, and the response carries the shelf as the
            # store now holds it.
            data = self._body()
            slug = str(data.get("case") or "")
            act = str(data.get("act") or "list")
            if act == "add":
                card = data.get("card") if isinstance(data.get("card"), dict) else {}
                ref, note, err = case_shelf_add(slug, card, by="operator")
                if err:
                    self._send(200, {"ok": False, "error": err})
                else:
                    self._send(200, {"ok": True, "note": note, "ref": ref,
                                     "shelf": case_shelf(slug),
                                     "chatId": (case_chat_for(slug) or {}).get("id")})
            elif act == "remove":
                gone, err = case_shelf_remove(slug, str(data.get("ref") or ""))
                if err:
                    self._send(200, {"ok": False, "error": err})
                else:
                    self._send(200, {"ok": True, "removed": gone,
                                     "shelf": case_shelf(slug)})
            else:
                if _case_manifest(slug) is None:
                    self._send(200, {"ok": False, "error": "No such case."})
                else:
                    self._send(200, {"ok": True, "shelf": case_shelf(slug),
                                     "chatId": (case_chat_for(slug) or {}).get("id")})
        elif self.path == "/api/cases/draft":
            # v0.4.5e (#241): Route B's swipe — a bridge-side
            # model pass over the chat's stored history proposes
            # the new case's form. Failures are named; the form
            # opens blank on any of them rather than guess.
            data = self._body()
            draft, err = case_draft_from_chat(data.get("chatId"), load_config())
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "draft": draft})
        elif self.path == "/api/cases/delete":
            # Build 15 (#163): delete is complete — the case, all
            # its Vault files, and its tied chat. The dashboard's
            # confirm names all of it (and the backup truth)
            # before this is ever called.
            data = self._body()
            gone, err = case_delete(str(data.get("case") or ""))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "deleted": gone})
        elif self.path == "/api/wire/start-case":
            # v0.6.6e (#330): the Wire card's Start a case
            # button — the same one act as the words door,
            # through case_birth_from_wire. The response
            # carries the birth's own accounting: leads filed,
            # gaps named, related shelved, tasks staged.
            data = self._body()
            result, err = case_birth_from_wire(str(data.get("link") or ""))
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, {"ok": True, "birth": result})
        elif self.path == "/api/wire/refresh":
            self._send(200, WIRE.refresh_now())
        elif self.path == "/api/wire/settings":
            data = self._body()
            cfg2 = load_config()
            block = cfg2.get("wire") if isinstance(cfg2.get("wire"), dict) else {}
            if "enabled" in data:
                block["enabled"] = data.get("enabled") is not False
            if "intervalMinutes" in data:
                block["intervalMinutes"] = _mesh_int(data.get("intervalMinutes"), 30)
            if "retentionDays" in data:
                block["retentionDays"] = _mesh_int(data.get("retentionDays"), 7)
            cfg2["wire"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "wire": wire_settings(cfg2)})
        elif self.path == "/api/wire/feeds":
            # Operator feeds (Build 3, #210): validated at add
            # time — the feed is fetched and parsed BEFORE it is
            # saved, and a refusal names its reason. A saved feed
            # is first-class from the next pull.
            data = self._body()
            name = str(data.get("name") or "").strip()[:80]
            url = str(data.get("url") or "").strip()[:500]
            if not name:
                self._send(200, {"ok": False, "error": "Give the feed a name first."})
            else:
                if wire_mod is None:
                    ok, sentence, count = False, "The wire engine is not loaded in this bridge.", 0
                else:
                    ok, sentence, count = wire_mod.validate_feed(url)
                if not ok:
                    self._send(200, {"ok": False, "error": sentence})
                else:
                    cfg2 = load_config()
                    block = cfg2.get("wire") if isinstance(cfg2.get("wire"), dict) else {}
                    feeds = wire_settings(cfg2)["feeds"]
                    fid = "custom-" + hashlib.sha1(url.encode("utf-8")).hexdigest()[:10]
                    if any(f["id"] == fid for f in feeds):
                        self._send(200, {"ok": False, "error": "That feed is already on the wire."})
                    else:
                        feeds.append({"id": fid, "name": name, "url": url})
                        block["feeds"] = feeds
                        cfg2["wire"] = block
                        save_config(cfg2)
                        self._send(200, {"ok": True, "wire": wire_settings(cfg2),
                                         "sentence": "Feed added — " + str(count) + " items on its current front page. It joins the next pull."})
        elif self.path == "/api/wire/feeds/remove":
            data = self._body()
            fid = str(data.get("id") or "")
            cfg2 = load_config()
            block = cfg2.get("wire") if isinstance(cfg2.get("wire"), dict) else {}
            feeds = [f for f in wire_settings(cfg2)["feeds"] if f["id"] != fid]
            block["feeds"] = feeds
            cfg2["wire"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "wire": wire_settings(cfg2)})
        elif self.path == "/api/pulse/create":
            # v0.5.13e (#287): the tab's deterministic door —
            # the same pulse_create the chat route receipts.
            data = self._body()
            with _pulse_lock:
                _watch, _err = pulse_create(
                    str(data.get("topic") or ""),
                    data.get("terms") if isinstance(data.get("terms"), list) else None,
                    str(data.get("case") or ""))
            if _err:
                self._send(200, {"ok": False, "error": _err})
            else:
                self._send(200, {"ok": True, "watch": _watch,
                                 "pulse": pulse_payload(load_config())})
        elif self.path == "/api/pulse/state":
            data = self._body()
            _watch, _err = pulse_set_state(str(data.get("id") or ""),
                                           str(data.get("state") or ""))
            if _err:
                self._send(200, {"ok": False, "error": _err})
            else:
                self._send(200, {"ok": True, "pulse": pulse_payload(load_config())})
        elif self.path == "/api/pulse/remove":
            data = self._body()
            _watch, _err = pulse_remove(str(data.get("id") or ""))
            if _err:
                self._send(200, {"ok": False, "error": _err})
            else:
                self._send(200, {"ok": True, "pulse": pulse_payload(load_config())})
        elif self.path == "/api/pulse/attach":
            data = self._body()
            _watch, _err = pulse_attach(str(data.get("id") or ""),
                                        str(data.get("case") or ""))
            if _err:
                self._send(200, {"ok": False, "error": _err})
            else:
                self._send(200, {"ok": True, "pulse": pulse_payload(load_config())})
        elif self.path == "/api/pulse/run":
            # A check on demand — the same pulse_run the
            # cadence loop and the greeting fold-in call.
            self._send(200, {"ok": True,
                             "summary": pulse_run(load_config(), reason="manual"),
                             "pulse": pulse_payload(load_config())})
        elif self.path == "/api/day":
            # v0.6.0e: the Day surface's deterministic door —
            # capture / ack / delete / remind / order / brief /
            # proposal, the same machinery chat receipts.
            data = self._body()
            self._send(200, daybook_act(data, load_config()))
        elif self.path == "/api/ctf/refresh":
            self._send(200, CTF.refresh_now())
        elif self.path == "/api/ctf/settings":
            data = self._body()
            cfg2 = load_config()
            block = cfg2.get("ctf") if isinstance(cfg2.get("ctf"), dict) else {}
            if "enabled" in data:
                block["enabled"] = data.get("enabled") is not False
            if "intervalMinutes" in data:
                block["intervalMinutes"] = _mesh_int(data.get("intervalMinutes"), 60)
            cfg2["ctf"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "ctf": ctf_settings(cfg2)})
        elif self.path == "/api/bounties/refresh":
            self._send(200, BOUNTIES.refresh_now())
        elif self.path == "/api/bounties/settings":
            data = self._body()
            cfg2 = load_config()
            block = cfg2.get("bounties") if isinstance(cfg2.get("bounties"), dict) else {}
            if "enabled" in data:
                block["enabled"] = data.get("enabled") is not False
            if "intervalMinutes" in data:
                block["intervalMinutes"] = _mesh_int(data.get("intervalMinutes"), 360)
            cfg2["bounties"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "bounties": bounties_settings(cfg2)})
        elif self.path == "/api/intel/refresh":
            # Build 15 (#165): a manual cycle, run on the engine's
            # own thread — the response never waits on a feed.
            self._send(200, INTEL.refresh_now())
        elif self.path == "/api/intel/watchlist":
            data = self._body()
            terms = data.get("terms")
            if not isinstance(terms, list):
                self._send(200, {"ok": False, "error": "The watchlist rides as a list of terms."})
            else:
                cfg2 = load_config()
                block = cfg2.get("intel") if isinstance(cfg2.get("intel"), dict) else {}
                block["watchlist"] = [str(t).strip() for t in terms if str(t).strip()][:40]
                cfg2["intel"] = block
                save_config(cfg2)
                try:
                    INTEL.compute_matches()
                except Exception:
                    pass
                self._send(200, {"ok": True, "watchlist": intel_settings(cfg2)["watchlist"]})
        elif self.path == "/api/intel/settings":
            data = self._body()
            cfg2 = load_config()
            block = cfg2.get("intel") if isinstance(cfg2.get("intel"), dict) else {}
            if "enabled" in data:
                block["enabled"] = data.get("enabled") is not False
            if "collectorPeer" in data:
                block["collectorPeer"] = str(data.get("collectorPeer") or "").strip()
            if "fallbackLocal" in data:
                block["fallbackLocal"] = data.get("fallbackLocal") is True
            if "cadenceHours" in data:
                block["cadenceHours"] = _mesh_int(data.get("cadenceHours"), 6)
            if isinstance(data.get("sources"), dict):
                src = block.get("sources") if isinstance(block.get("sources"), dict) else {}
                for k in ("nvd", "kev", "epss", "cisa", "rss", "edb"):
                    if k in data["sources"]:
                        src[k] = data["sources"][k] is not False
                block["sources"] = src
            cfg2["intel"] = block
            save_config(cfg2)
            self._send(200, {"ok": True, "intel": intel_settings(cfg2)})
        elif self.path == "/api/intel/search":
            # v0.5.2e (#259): one search over the local catalog —
            # CVE id exact/prefix, free text, vendor/product
            # terms; KEV-only, matched-only, and date-window
            # filters. The catalog's last-fetched stamp rides the
            # result, so a search over a stale catalog says so on
            # its face. Local only: no live calls per keystroke.
            data = self._body()
            try:
                _days = int(data.get("days") or 0)
            except Exception:
                _days = 0
            try:
                self._send(200, INTEL.search(
                    str(data.get("q") or ""),
                    kev_only=data.get("kevOnly") is True,
                    matched_only=data.get("matchedOnly") is True,
                    days=_days or None,
                    limit=data.get("limit") or 25))
            except Exception as e:
                self._send(200, {"ok": False, "error": "The intel store could not be searched: " + str(e)[:160]})
        elif self.path == "/api/intel/file":
            # Build 15 (#165 × #161): file an intel record into a
            # case as a finding. The citation travels in the note
            # itself — CVE ID and the digest's stamp — so case
            # exports carry it without the Center existing there.
            data = self._body()
            slug = str(data.get("case") or "")
            cid = str(data.get("cve") or "").upper()
            item = None
            try:
                item = INTEL.item(cid)
            except Exception:
                item = None
            if item is None:
                self._send(200, {"ok": False, "error": "That record is not in the local intel store."})
            elif _case_manifest(slug) is None:
                self._send(200, {"ok": False, "error": "No such case."})
            else:
                gen_at = ""
                coll = ""
                try:
                    with open(INTEL.digest_path, encoding="utf-8") as f:
                        _dg = json.load(f)
                    gen_at = str(_dg.get("generated_at") or "")
                    coll = str((_dg.get("collector") or {}).get("name") or "")
                except Exception:
                    gen_at = ""
                cvss = item.get("cvss")
                sev = "info"
                if isinstance(cvss, (int, float)):
                    sev = "critical" if cvss >= 9 else "high" if cvss >= 7 else "medium" if cvss >= 4 else "low"
                body_lines = [
                    item.get("title") or cid, "",
                    item.get("description") or "", "",
                    "- **CVE:** " + cid,
                    "- **CVSS:** " + (str(cvss) if cvss is not None else "—") + ((" (" + item["cvssSeverity"] + ")") if item.get("cvssSeverity") else ""),
                    "- **EPSS:** " + (str(item["epss"]) if item.get("epss") is not None else "—"),
                    "- **Known-exploited (CISA KEV):** " + ("yes — remediation due " + (item.get("kevDue") or "?") if item.get("kev") else "no"),
                    "- **Products (CPE):** " + (", ".join(item.get("products") or []) or "—"),
                    "",
                    "Filed from The Wire — " + cid
                    + ", per the intel digest generated " + (gen_at or "at an unstamped time")
                    + ((" by " + coll) if coll else "")
                    + ". Sources: NVD / CISA KEV / EPSS, as stamped in the Center.",
                ]
                exs = item.get("exploits") or []
                if exs:
                    e0 = exs[0]
                    total_ex = item.get("exploitCount") or len(exs)
                    body_lines += ["", "Public exploit code: EDB-" + str(e0.get("edb"))
                                   + " (" + str(e0.get("type") or "exploit") + " · "
                                   + str(e0.get("platform") or "unknown platform") + ")"
                                   + ((" — plus " + str(total_ex - 1) + " more in the index")
                                      if total_ex > 1 else "")
                                   + " — Exploit-DB, mirrored locally and joined by CVE id."]
                refs = item.get("refs") or []
                if refs:
                    body_lines += ["", "References: " + " · ".join(refs[:5])]
                filed, err = case_file_entry(slug, "finding", "Intel — " + cid, "\n".join(body_lines), sev, False)
                if err:
                    self._send(200, {"ok": False, "error": err})
                else:
                    self._send(200, {"ok": True, "filed": filed})
        elif self.path == "/api/github/badge-sync/start":
            # #136: slotting the Operator Badge pulls the latest
            # backup into a preview on a live operation card. The
            # badge authenticates nothing; the sign-in is the key.
            data = self._body()
            dash = data.get("dashboard") if isinstance(data.get("dashboard"), dict) else None
            self._send(200, badge_sync_start(load_config(), dash))
        elif self.path == "/api/github/badge-sync/apply":
            data = self._body()
            self._send(200, badge_sync_apply(load_config(), str(data.get("operationId") or "")))
        elif self.path == "/api/github/badge-sync/dismiss":
            data = self._body()
            self._send(200, badge_sync_dismiss(str(data.get("operationId") or "")))
        elif self.path == "/api/sync/run":
            # Build 3 (#176): the Sync control's round — pull,
            # converge, push — riding the Operator identity. The
            # open thread's id rides along so the card can say
            # whether the thread on screen was in the round.
            # v0.3.0e Build 1 (#200): the page's seat state rides
            # along too, and the converged seat state answers, so
            # the window adopts what the round settled.
            data = self._body()
            if isinstance(data.get("seat"), dict):
                _seat_merge(data.get("seat"), source="round")
            # v0.6.2e (#303): a directional press runs its
            # one-way round; no direction is the legacy round.
            result = sync_override_round(
                load_config(), str(data.get("direction") or ""),
                "button", str(data.get("openChatId") or ""))
            if isinstance(result, dict):
                result["seat"] = seat_state_payload()
            self._send(200, result)
        elif self.path == "/api/sync/seat":
            # v0.3.0e Build 1 (#200): a seat notes itself. The
            # page sends its seat state as a window opens (and as
            # it changes); the bridge merges it, notes the session
            # under the same gap/debounce discipline the DHD door
            # uses — which is what fires the refresh pull for a
            # long-lived bridge — and answers with the converged
            # seat state and the sync state's clock, so the page
            # can adopt and re-read when a pull lands.
            data = self._body()
            if isinstance(data.get("seat"), dict):
                _seat_merge(data.get("seat"))
            sync_note_app_session(data.get("seatKey"))
            self._send(200, {"ok": True, "seat": seat_state_payload(),
                             "sync": sync_state_payload()})
        elif self.path == "/api/github/backup":
            data = self._body()
            dash = data.get("dashboard") if isinstance(data.get("dashboard"), dict) else None
            self._send(200, github_run_backup(load_config(), dash))
        elif self.path == "/api/github/import/preview":
            # Preview writes nothing: it pulls the backup down and
            # reports, per essential, what restoring it would do.
            data = self._body()
            dash = data.get("dashboard") if isinstance(data.get("dashboard"), dict) else None
            fetched, err = github_fetch_backup(load_config())
            if err:
                self._send(200, {"ok": False, "error": err, "changes": []})
            else:
                changes, _items = github_import_changes(load_config(), fetched, dash)
                self._send(200, {
                    "ok": True,
                    "error": None,
                    "changes": changes,
                    "backedUpAt": fetched["manifest"].get("backedUpAt"),
                })
        elif self.path == "/api/github/import/apply":
            # Only ever the second half of preview → confirm. The
            # apply re-fetches, so what lands is the backup as it
            # stands now, and the dashboard settings in the backup
            # ride back in the answer for the dashboard to adopt.
            fetched, err = github_fetch_backup(load_config())
            if err:
                self._send(200, {"ok": False, "error": err})
            else:
                self._send(200, github_apply_import(load_config(), fetched))
        else:
            self._send(404, {"error": "not found"})

    def log_message(self, *args):
        pass

