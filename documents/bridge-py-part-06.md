# bridge.py — part 6 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 20498–24856 of bridge.py.
Top-level symbols in this part: execute_chip, load_workflows, save_workflows, run_workflow_chain, load_rag_installs, save_rag_installs, _rag_embed, _rag_chunks, _pdf_parse_cmap, _pdf_obj_stream, _pdf_font_maps, _pdf_decode_coded, _pdf_decode_bytes, _pdf_literal_bytes, _pdf_content_text, pdf_extract_text, build_rag_index, write_rag_pack, build_rag_repo_files, rag_readme_override, rag_repo_files, export_rag_pack, build_chip_repo_files, load_local_chip, save_local_chip_readme, export_chip, github_publish_files, _rag_pack_index, build_rag_cartridge_png, parse_rag_cartridge_png, rag_cartridge_payload, install_rag_cartridge, local_rag_packs, rag_pack_documents, _vault_seed_files, vault_join, vault_fs_path, vault_write, vault_raw_payload, vault_delete_note, vault_delete_folder, vault_ensure_seed, _vault_links, _vault_tags, _vault_is_case_fragment, vault_notes_raw, vault_list_payload, vault_search, vault_file_from_chat, rag_retrieve, strip_citation_footer, fetch_pack_documents, rag_install_record, rag_pack_slug, rag_catalog_view, _adopt_voice_venv, VoiceUnavailable, voice_config, save_voice_config, _module_available, _probe_engine, _engine_detail, _voice_decode_probe, _decode_detail, _voice_cuda_probe, _cuda_fallback_sentence, _voice_stage_devices, _kokoro_device, _is_cuda_error, _voice_cuda_mark_failed, _unavailable_message, _kokoro_voice_label, discover_kokoro_voices, discover_piper_voices, voice_status, load_voice_state, save_voice_state, load_voice_calls, _stt_model, transcribe_audio.


def execute_chip(chip, args_text, cfg):
    # Runs a chip the way Navi does: its run(arguments) with a working
    # navi_internal, in a subprocess with a 20-second budget. A
    # chip whose code defines no run() is declarative — the
    # output it declares is its result. When real execution fails,
    # the declared output is shown with the reason, never swallowed.
    code = chip.get("code") or ""
    declared = chip.get("output") or ""
    label = "chip " + chip["slug"]
    if "def run" not in code:
        if declared:
            return {"command": label, "output": declared, "exit_code": 0}
        return {
            "command": label,
            "output": "(chip \"" + chip["name"] + "\" declares no output and defines no run() to execute.)",
            "exit_code": 0,
        }
    endpoint = cfg.get("live", {}).get("llamafile_endpoint", "http://localhost:8080/v1")
    model = cfg.get("live", {}).get("model_name") or "navi"
    tmp = tempfile.mkdtemp(prefix="navi-chip-")
    try:
        with open(os.path.join(tmp, "navi_internal.py"), "w") as f:
            f.write(NAVI_INTERNAL_STUB)
        # The navi.py compatibility shim (build .86, issue #79)
        # rides EVERY sandbox — unless the chip ships its own
        # navi.py, which always wins.
        chip_files_pre = chip.get("files") or []
        if not any(isinstance(it, dict) and str(it.get("path") or "") == "navi.py" for it in chip_files_pre):
            with open(os.path.join(tmp, "navi.py"), "w") as f:
                f.write(NAVI_PY_SHIM)
        # The chip's whole file set lands in the sandbox: the entry
        # code as root.py (and chip_module.py, the name the runner
        # imports) plus every module file at its own path, so
        # root.py's imports resolve exactly as authored. Paths are
        # re-validated here — a chip can come from anywhere.
        chip_files = chip.get("files") or []
        for item in chip_files:
            if not isinstance(item, dict):
                continue
            rel = str(item.get("path") or "")
            parts = rel.split("/")
            if (not rel or rel.startswith("/") or ".." in parts
                    or not rel.endswith(".py") or rel == "navi_internal.py"
                    or rel == "root.py"):
                continue
            dest = os.path.join(tmp, rel)
            os.makedirs(os.path.dirname(dest), exist_ok=True)
            with open(dest, "w") as f:
                f.write(str(item.get("content") or ""))
        with open(os.path.join(tmp, "root.py"), "w") as f:
            f.write(code)
        # Package mode (build .86, issue #79): when the entry
        # code uses relative imports, or the file set carries
        # package directories, the entry loads as a package
        # member — it is written into a synthetic package with
        # the chip's package folders beside it, and the runner
        # imports it as chip_pkg.chip_entry, so a relative
        # import (from .app_cryptex...) resolves, and a plain
        # "import app_cryptex..." finds the package too — the
        # package dir itself joins the path.
        file_paths = [str(it.get("path") or "") for it in chip_files if isinstance(it, dict)]
        package_mode = (
            re.search(r"^\s*from\s+\." , code, re.M) is not None
            or any("/" in p for p in file_paths)
        )
        if package_mode:
            pkg_dir = os.path.join(tmp, "chip_pkg")
            os.makedirs(pkg_dir, exist_ok=True)
            with open(os.path.join(pkg_dir, "__init__.py"), "w") as f:
                f.write("")
            with open(os.path.join(pkg_dir, "chip_entry.py"), "w") as f:
                f.write(code)
            for item in chip_files:
                if not isinstance(item, dict):
                    continue
                rel = str(item.get("path") or "")
                if "/" not in rel or rel.startswith("/") or ".." in rel.split("/"):
                    continue
                dest = os.path.join(pkg_dir, rel)
                os.makedirs(os.path.dirname(dest), exist_ok=True)
                with open(dest, "w") as f:
                    f.write(str(item.get("content") or ""))
            runner = (
                "import sys, types\n"
                "sys.path.insert(0, sys.argv[1])\n"
                "sys.path.insert(0, sys.argv[1] + \"/chip_pkg\")\n"
                "import chip_pkg.chip_entry as chip_module\n"
                "args = types.SimpleNamespace(text=sys.argv[2])\n"
                "chip_module.run(args)\n"
            )
        else:
            with open(os.path.join(tmp, "chip_module.py"), "w") as f:
                f.write(code)
            runner = (
                "import sys, types\n"
                "sys.path.insert(0, sys.argv[1])\n"
                "import chip_module\n"
                "args = types.SimpleNamespace(text=sys.argv[2])\n"
                "chip_module.run(args)\n"
            )
        with open(os.path.join(tmp, "runner.py"), "w") as f:
            f.write(runner)
        env = dict(os.environ)
        env["NAVI_LLAMAFILE_ENDPOINT"] = endpoint
        env["NAVI_MODEL"] = model
        env["PYTHONPATH"] = tmp
        try:
            proc = subprocess.run(
                [sys.executable, os.path.join(tmp, "runner.py"), tmp, args_text or ""],
                capture_output=True, text=True, timeout=20, cwd=shell_cwd(), env=env,
            )
            out = ((proc.stdout or "") + (proc.stderr or "")).strip()
            if proc.returncode == 0:
                return {"command": label, "output": out or "(chip ran — no output)", "exit_code": 0}
            note = "The chip's code exited with code " + str(proc.returncode) + "."
            if declared:
                return {
                    "command": label,
                    "output": declared + "\n\n(" + note + " Showing the output the chip declares instead. Error tail:\n" + out[-400:] + ")",
                    "exit_code": 0,
                }
            return {"command": label, "output": note + "\n" + out[-1200:], "exit_code": proc.returncode}
        except subprocess.TimeoutExpired:
            msg = "The chip ran longer than 20 seconds and was stopped."
            if declared:
                return {
                    "command": label,
                    "output": declared + "\n\n(" + msg + " Showing the output the chip declares instead.)",
                    "exit_code": 0,
                }
            return {"command": label, "output": msg, "exit_code": 1}
    finally:
        shutil.rmtree(tmp, ignore_errors=True)


# ------------------------------ Workflows --------------------------------
# Saved chains of chips (chips/workflows.json): each chip's
# output becomes the next chip's input — execute_chip receives the
# incoming output as its arguments text, the way a following chip in
# a pipeline would read what came before it.

WORKFLOWS_FILE = os.path.join(CHIPS_DIR, "workflows.json")


def load_workflows():
    try:
        with open(WORKFLOWS_FILE) as f:
            rows = json.load(f)
        return rows if isinstance(rows, list) else []
    except Exception:
        return []


def save_workflows(rows):
    os.makedirs(CHIPS_DIR, exist_ok=True)
    with open(WORKFLOWS_FILE, "w") as f:
        json.dump(rows, f, indent=1)


def run_workflow_chain(wf, cfg):
    by_file = {c["file"]: c for c in local_chips()}
    steps = []
    incoming = ""
    for fn in wf.get("steps", []):
        chip = by_file.get(fn)
        if chip is None:
            return {
                "ok": False,
                "error": "Step chip \"" + fn + "\" is no longer in the chips folder — the chain stops here.",
                "steps": steps,
                "finalOutput": None,
            }
        result = execute_chip(chip, incoming, cfg)
        steps.append({
            "chipId": fn,
            "name": chip["name"],
            "input": incoming,
            "output": result["output"],
        })
        incoming = result["output"]
    return {"ok": True, "error": None, "steps": steps, "finalOutput": incoming or None}


# --------------------- Knowledge packs (RAG) ---------------------
# Her knowledge, chip-style like her chips. A pack is a GitHub
# repo carrying the "navi-rag" topic; its documents are the repo's
# .md/.txt/.pdf files — a PDF's text is extracted at the door
# (pdf_extract_text below, stdlib only) and only the text is ever
# stored or indexed. Installing downloads the documents into
# rag-packs/<slug>/, builds the pack's index there at install time,
# and records the install in rag-packs/installed.json with an
# active flag — only active packs feed retrieval in /api/chat, and
# when a pack informs an answer, its sources ride the reply's "rag"
# payload field and the dashboard draws them as pills under that
# message — never as words in her reply. RAG Studio forges packs
# from documents on
# this machine into the same folder with local/ records. A pack
# also exports back out as a GitHub-ready repo ZIP (README page,
# navi-rag.json manifest, documents/, and the built index.json as
# the pack's reference copy — installs rebuild the index locally
# from the documents and skip a root README, so the page never
# pollutes retrieval). The index
# is feature-hashed term vectors (RAG_EMBED_DIM buckets, signed,
# L2-normalized): small local embeddings computed with the stdlib
# alone — no model to download, a chunk embeds in well under a
# millisecond, and at query time only the question gets embedded,
# so retrieval stays fluid on CPU. Packs and their indexes sit on
# the installers' persistence keep-list beside chips/ and
# voice/models/, so updates and --fresh installs keep them.

RAG_PACKS_DIR = os.path.join(ROOT, "rag-packs")
RAG_EXPORTS_DIR = os.path.join(ROOT, "rag-exports")
CHIPS_EXPORTS_DIR = os.path.join(ROOT, "chips-exports")
RAG_INSTALLS_PATH = os.path.join(RAG_PACKS_DIR, "installed.json")
RAG_EMBED_DIM = 256
RAG_MIN_SCORE = 0.18
RAG_MAX_DOCS = 40
RAG_MAX_DOC_CHARS = 200000


def load_rag_installs():
    try:
        with open(RAG_INSTALLS_PATH) as f:
            rows = json.load(f)
        return rows if isinstance(rows, list) else []
    except Exception:
        return []


def save_rag_installs(rows):
    os.makedirs(RAG_PACKS_DIR, exist_ok=True)
    with open(RAG_INSTALLS_PATH, "w") as f:
        json.dump(rows, f, indent=2)


def _rag_embed(text):
    # A feature-hashed term embedding: each word hashes into one of
    # RAG_EMBED_DIM buckets (a second bit of the hash picks the
    # sign, so unrelated words sharing a bucket tend to cancel
    # rather than reinforce), counts accumulate, and the vector is
    # L2-normalized so a dot product is a cosine.
    vec = [0.0] * RAG_EMBED_DIM
    for tok in re.findall(r"[a-z0-9]+", (text or "").lower()):
        if len(tok) < 3:
            continue
        h = zlib.crc32(tok.encode("utf-8"))
        vec[h % RAG_EMBED_DIM] += 1.0 if (h >> 9) & 1 else -1.0
    norm = math.sqrt(sum(v * v for v in vec))
    if norm:
        vec = [v / norm for v in vec]
    return vec


def _rag_chunks(body):
    # Paragraphs, with long ones split into sentence groups — the
    # same chunking the bench mirror uses, small enough that a
    # retrieved passage reads as one thought.
    text = (body or "").replace("\r\n", "\n")
    chunks = []
    for para in text.split("\n\n"):
        para = " ".join(para.split())
        if not para:
            continue
        if len(para) <= 420:
            chunks.append(para)
            continue
        cur = ""
        for sent in para.split(". "):
            s = sent.strip()
            if not s:
                continue
            if cur and len(cur) + len(s) + 2 > 420:
                chunks.append(cur)
                cur = s
            else:
                cur = (cur + ". " + s) if cur else s
        if cur:
            chunks.append(cur)
    return chunks


def _pdf_parse_cmap(content):
    # A ToUnicode CMap -> {code length: {code: unicode text}},
    # from its beginbfchar / beginbfrange sections.
    maps = {}

    def dst_text(hx):
        try:
            return bytes.fromhex(hx).decode("utf-16-be", "replace")
        except Exception:
            return ""

    for sec in re.finditer(rb"beginbfchar(.*?)endbfchar", content, re.S):
        for m in re.finditer(rb"<([0-9A-Fa-f]+)>\s*<([0-9A-Fa-f]+)>", sec.group(1)):
            src = m.group(1).decode("ascii")
            maps.setdefault(len(src) // 2, {})[int(src, 16)] = dst_text(m.group(2).decode("ascii"))
    for sec in re.finditer(rb"beginbfrange(.*?)endbfrange", content, re.S):
        body = sec.group(1)
        for m in re.finditer(rb"<([0-9A-Fa-f]+)>\s*<([0-9A-Fa-f]+)>\s*\[(.*?)\]", body, re.S):
            lo = int(m.group(1), 16)
            bucket = maps.setdefault(len(m.group(1)) // 2, {})
            for k, d in enumerate(re.findall(rb"<([0-9A-Fa-f]+)>", m.group(3))):
                bucket[lo + k] = dst_text(d.decode("ascii"))
        plain = re.sub(rb"<[0-9A-Fa-f]+>\s*<[0-9A-Fa-f]+>\s*\[.*?\]", b"", body, flags=re.S)
        for m in re.finditer(rb"<([0-9A-Fa-f]+)>\s*<([0-9A-Fa-f]+)>\s*<([0-9A-Fa-f]+)>", plain):
            lo = int(m.group(1), 16)
            hi = int(m.group(2), 16)
            base = int(m.group(3), 16)
            if hi - lo > 4096:
                continue
            bucket = maps.setdefault(len(m.group(1)) // 2, {})
            for code in range(lo, hi + 1):
                v = base + (code - lo)
                bucket[code] = chr(v) if 0 <= v < 0x110000 else ""
    return maps


def _pdf_obj_stream(body):
    # The inflated stream bytes of an indirect object body.
    i = body.find(b"stream")
    if i < 0:
        return None
    start = i + 6
    if body[start:start + 2] == b"\r\n":
        start += 2
    elif body[start:start + 1] in (b"\n", b"\r"):
        start += 1
    else:
        return None
    end = body.find(b"endstream", start)
    if end < 0:
        return None
    raw = body[start:end]
    if raw.endswith(b"\r\n"):
        raw = raw[:-2]
    elif raw.endswith(b"\n") or raw.endswith(b"\r"):
        raw = raw[:-1]
    try:
        return zlib.decompress(raw)
    except Exception:
        return raw


def _pdf_font_maps(data):
    # {resource name (F1, F2, ...): ToUnicode map} for the fonts
    # that carry one, plus the document's only map when there is
    # exactly one (the fallback for a stream whose font name did
    # not resolve). Font dictionaries and resource maps are read
    # from the uncompressed indirect objects; ToUnicode CMaps are
    # streams, so they are never buried in object streams.
    objs = {}
    for m in re.finditer(rb"(\d+)\s+\d+\s+obj\b(.*?)endobj", data, re.S):
        try:
            objs[int(m.group(1))] = m.group(2)
        except Exception:
            pass
    cmaps = {}
    for num, body in objs.items():
        m = re.search(rb"/ToUnicode\s+(\d+)\s+\d+\s+R", body)
        if m:
            stream = _pdf_obj_stream(objs.get(int(m.group(1)), b""))
            if stream:
                cmap = _pdf_parse_cmap(stream)
                if cmap:
                    cmaps[num] = cmap
    by_name = {}
    for body in objs.values():
        for sec in re.finditer(rb"/Font\s*<<(.*?)>>", body, re.S):
            for m in re.finditer(rb"/([A-Za-z0-9]+)\s+(\d+)\s+\d+\s+R", sec.group(1)):
                ref = int(m.group(2))
                if ref in cmaps:
                    by_name[m.group(1).decode("latin-1")] = cmaps[ref]
    only = next(iter(cmaps.values())) if len(cmaps) == 1 else None
    return by_name, only


def _pdf_decode_coded(raw, cmap):
    # Decodes one shown string's raw bytes through a font's
    # ToUnicode map (longest codes first); without a map — or for
    # a code the map does not know in a 1-byte font — the plain
    # byte decoding is the fallback.
    if cmap is None:
        return _pdf_decode_bytes(raw)
    lengths = sorted(cmap.keys(), reverse=True)
    out = []
    i = 0
    n = len(raw)
    while i < n:
        matched = False
        for ln in lengths:
            if i + ln <= n:
                code = int.from_bytes(raw[i:i + ln], "big")
                if code in cmap[ln]:
                    out.append(cmap[ln][code])
                    i += ln
                    matched = True
                    break
        if not matched:
            if lengths and lengths[-1] == 1:
                out.append(_pdf_decode_bytes(raw[i:i + 1]))
            i += 1
    return "".join(out)


def _pdf_decode_bytes(raw):
    # A PDF string's bytes: a FE FF BOM means UTF-16BE; otherwise
    # try UTF-8 and fall back to latin-1, which never loses a byte.
    if raw[:2] == bytes([254, 255]):
        return raw[2:].decode("utf-16-be", "replace")
    try:
        return raw.decode("utf-8")
    except Exception:
        return raw.decode("latin-1", "replace")


def _pdf_literal_bytes(content, i):
    # content[i] is '('. Reads the literal string (escapes, octal
    # codes, balanced nested parens); returns (raw bytes, next
    # index) — decoding waits for the active font's ToUnicode map.
    out = bytearray()
    depth = 1
    i += 1
    n = len(content)
    simple = {110: 10, 114: 13, 116: 9, 98: 8, 102: 12, 40: 40, 41: 41, 92: 92}
    while i < n and depth:
        c = content[i]
        if c == 92:  # backslash escape
            i += 1
            if i >= n:
                break
            e = content[i]
            if e in simple:
                out.append(simple[e])
                i += 1
            elif 48 <= e <= 55:  # up to three octal digits
                j = i
                val = 0
                while j < n and j < i + 3 and 48 <= content[j] <= 55:
                    val = val * 8 + (content[j] - 48)
                    j += 1
                out.append(val & 255)
                i = j
            elif e in (10, 13):  # line continuation
                if e == 13 and i + 1 < n and content[i + 1] == 10:
                    i += 1
                i += 1
            else:
                out.append(e)
                i += 1
        elif c == 40:
            depth += 1
            out.append(c)
            i += 1
        elif c == 41:
            depth -= 1
            if depth:
                out.append(c)
            i += 1
        else:
            out.append(c)
            i += 1
    return bytes(out), i


def _pdf_content_text(content, font_maps, default_cmap):
    # Walks one decompressed content stream: literal and hex
    # strings are operands, held as raw bytes; a text-showing
    # operator (Tj, TJ, ', ") decodes them through the active
    # font's ToUnicode map (Tf selects the font), and position
    # moves (Td, TD, T*) end the current line. TJ kerning numbers
    # below -250 read as the word gaps they are.
    lines = []
    cur = []
    operands = []
    cmap = default_cmap
    i = 0
    n = len(content)
    while i < n:
        c = content[i]
        if c == 40:  # ( literal string
            raw, i = _pdf_literal_bytes(content, i)
            operands.append(("s", raw))
        elif c == 60 and content[i + 1:i + 2] != b"<":  # < hex string
            j = content.find(b">", i)
            if j < 0:
                break
            hx = re.sub(rb"\s", b"", content[i + 1:j])
            if len(hx) % 2:
                hx += b"0"
            try:
                operands.append(("s", bytes.fromhex(hx.decode("ascii"))))
            except Exception:
                pass
            i = j + 1
        elif c == 47:  # /name — the font operand of Tf
            j = i + 1
            while j < n and content[j] not in (32, 9, 13, 10, 12, 0) and content[j] not in b"()<>[]{}/%":
                j += 1
            operands.append(("name", content[i + 1:j].decode("latin-1")))
            i = j
        elif c in (32, 9, 13, 10, 12, 0):
            i += 1
        else:
            j = i
            while j < n and content[j] not in (32, 9, 13, 10, 12, 0) and content[j] not in b"()<>[]{}/%":
                j += 1
            if j == i:
                i += 1
                continue
            word = content[i:j]
            i = j
            if word in (b"Tj", b"'", b'"'):
                for kind, val in operands:
                    if kind == "s":
                        cur.append(_pdf_decode_coded(val, cmap))
                cur.append(" ")
                operands = []
            elif word == b"TJ":
                for kind, val in operands:
                    if kind == "s":
                        cur.append(_pdf_decode_coded(val, cmap))
                    elif kind == "n" and val <= -250:
                        cur.append(" ")
                cur.append(" ")
                operands = []
            elif word in (b"Td", b"TD", b"T*", b"ET"):
                if cur:
                    lines.append("".join(cur))
                    cur = []
                operands = []
            elif word == b"Tf":
                for kind, val in operands:
                    if kind == "name":
                        cmap = font_maps.get(val, default_cmap)
                operands = []
            else:
                try:
                    operands.append(("n", float(word)))
                except Exception:
                    operands = []
    if cur:
        lines.append("".join(cur))
    return lines


def pdf_extract_text(data):
    # The text layer of a PDF, stdlib only: every stream is
    # inflated (FlateDecode; a stream that does not inflate is
    # used raw) and the ones carrying text operators contribute
    # their strings in order, decoded through each font's
    # ToUnicode map when it carries one — the CID-font exports
    # Word and writing tools produce need exactly that, their
    # raw codes being font-internal numbers, not characters.
    # The playbooks this targets (CISA, NIST, vendor PDFs) use
    # standard encodings and extract cleanly; a scanned or
    # image-only PDF has no text layer and extracts to "",
    # which callers report instead of indexing silence.
    if not data or data[:4] != b"%PDF":
        return ""
    font_maps, default_cmap = _pdf_font_maps(data)
    lines = []
    pos = 0
    while True:
        s = data.find(b"stream", pos)
        if s < 0:
            break
        if data[s - 3:s] == b"end":  # the "stream" inside "endstream"
            pos = s + 6
            continue
        start = s + 6
        if data[start:start + 2] == b"\r\n":
            start += 2
        elif data[start:start + 1] in (b"\n", b"\r"):
            start += 1
        else:
            pos = s + 6
            continue
        end = data.find(b"endstream", start)
        if end < 0:
            break
        raw = data[start:end]
        if raw.endswith(b"\r\n"):
            raw = raw[:-2]
        elif raw.endswith(b"\n") or raw.endswith(b"\r"):
            raw = raw[:-1]
        try:
            content = zlib.decompress(raw)
        except Exception:
            content = raw
        pos = end + 9
        if b"Tj" not in content and b"TJ" not in content:
            continue
        lines.extend(_pdf_content_text(content, font_maps, default_cmap))
    text = "\n".join(lines)
    return "".join(ch for ch in text if ch in "\n\t" or ord(ch) >= 32).strip()


def build_rag_index(documents):
    # The index is built once, at install/forge time, and stored
    # with the pack; queries never re-embed the documents.
    docs = []
    total = 0
    for d in documents:
        chunks = [
            {"text": c, "vector": _rag_embed(c)}
            for c in _rag_chunks(d.get("body") or "")
        ]
        total += len(chunks)
        docs.append({"title": d.get("title") or "document", "chunks": chunks})
    return {"documents": docs, "chunkCount": total}


def write_rag_pack(slug, name, description, documents, icon=None):
    # The pack on disk: its documents as files plus index.json —
    # the built index and the pack metadata — beside them. A
    # rewrite (update, re-forge) replaces the documents wholesale.
    pdir = os.path.join(RAG_PACKS_DIR, slug)
    os.makedirs(pdir, exist_ok=True)
    for fn in os.listdir(pdir):
        if fn.lower().endswith((".md", ".txt", ".pdf")) or fn == "index.json":
            try:
                os.remove(os.path.join(pdir, fn))
            except Exception:
                pass
    used = set()
    for d in documents:
        flat = (d.get("title") or "document").replace("/", "-")
        safe = re.sub(r"[^A-Za-z0-9._-]+", "-", flat).strip("-.") or "document"
        if safe.lower().endswith(".pdf"):
            # A PDF document's body is its extracted text, so on
            # disk it lives as a .txt file under its own name —
            # the pack never stores the binary.
            safe = safe[:-4] + ".txt"
        elif not safe.lower().endswith((".md", ".txt")):
            safe += ".md"
        stem, dot, ext = safe.rpartition(".")
        cand = safe
        n = 2
        while cand in used:
            cand = (stem + "-" + str(n) + "." + ext) if dot else (safe + "-" + str(n))
            n += 1
        used.add(cand)
        with open(os.path.join(pdir, cand), "w") as f:
            f.write(d.get("body") or "")
    index = build_rag_index(documents)
    index["name"] = name
    index["description"] = description
    # Build 3 (#101): the pack's declared icon art lives on its
    # index record, the pack folder's metadata home — validated
    # here so a bad shape never reaches a face; the face falls
    # back to the book. Callers who mean to REFUSE bad art (the
    # Studio, the exports) validate before they get here.
    icon_rows = valid_icon_art(icon)
    if icon_rows:
        index["icon"] = icon_rows
    with open(os.path.join(pdir, "index.json"), "w") as f:
        json.dump(index, f)
    return index


def build_rag_repo_files(name, description, documents, version, index_text="", icon=None):
    # The GitHub-ready repo layout for a pack — the format the
    # installer already reads, made explicit. documents is
    # [{title, body}] with title a bare file name (the pack folder
    # stores documents flat, PDF text already as .txt). Returns
    # [(arcname, text)] in repo order: README page, manifest,
    # index.json (the pack's built index, shipped as the reference
    # copy — installing Navi rebuilds it locally from the
    # documents, so the shipped copy can never disagree),
    # documents/, .gitignore.
    doc_files = [d["title"] for d in documents]
    manifest = {
        "format": "navi-rag/1",
        "name": name,
        "description": description,
        "version": version,
        "documents": ["documents/" + f for f in doc_files],
        "index": "index.json",
        "generatedBy": "Apex — RAG Studio pack export",
    }
    # Build 3 (#101): the pack's declared icon art ships in its
    # manifest, the way a chip's does — malformed art refuses the
    # export in plain words rather than shipping a manifest whose
    # face silently drops it.
    if icon:
        if valid_icon_art(icon) is None:
            raise ValueError(ICON_ART_REFUSAL)
        manifest["icon"] = valid_icon_art(icon)
    lines = [
        "# " + name,
        "",
        description or "A knowledge pack for Apex.",
        "",
        "A **knowledge pack** for Apex — Navi's answers can draw on",
        "these documents, and cite the pack and document they came from.",
        "The pack format is `navi-rag/1` (see `navi-rag.json`).",
        "",
        "## Documents",
        "",
    ]
    for f in doc_files:
        lines.append("- `documents/" + f + "`")
    lines += [
        "",
        "## Install it in Navi",
        "",
        "1. Open Navi's **RAG** tab, then **Packs**.",
        "2. **Search GitHub** — every public repo tagged `navi-rag` is listed.",
        "3. **Install** this pack. Navi downloads the documents at the latest",
        "   release and builds the pack's index on your machine from them.",
        "",
        "Only **active** packs feed her answers; toggle the pack off in the",
        "same tab whenever you like.",
        "",
        "## Layout",
        "",
        "- `README.md` — this page. Navi's installer skips a root README, so",
        "  it describes the pack without ever being indexed as a document.",
        "- `navi-rag.json` — the pack manifest (name, version, document list).",
        "- `index.json` — the pack's built index, shipped as a reference copy;",
        "  installing Navi rebuilds it locally from the documents.",
        "- `documents/` — the pack's documents (`.md` / `.txt`).",
        "",
        "## Publishing an update",
        "",
        "Edit the documents, bump `version` in `navi-rag.json`, push, and publish",
        "a new release — the version tag is created with that publish (this export is `"
        + version + "`). A newer release is what offers everyone who",
        "installed the pack an update. Keep the `navi-rag` topic on the",
        "repo — that is how Navi finds it.",
        "",
    ]
    gitignore = chr(10).join([
        ".DS_Store",
        "Thumbs.db",
        "",
    ])
    files = [
        ("README.md", chr(10).join(lines)),
        ("navi-rag.json", json.dumps(manifest, indent=2) + chr(10)),
    ]
    if index_text:
        files.append(("index.json", index_text))
    for d in documents:
        files.append(("documents/" + d["title"], d["body"]))
    files.append((".gitignore", gitignore))
    return files


def rag_readme_override(slug):
    # The operator's edited README for a pack, kept in the pack
    # folder as a hidden file (no .md/.txt suffix, so it is never
    # collected as a document). None when they have not edited it —
    # the generated README ships instead.
    try:
        with open(os.path.join(RAG_PACKS_DIR, slug, ".readme-override")) as f:
            return f.read()
    except Exception:
        return None


def rag_repo_files(slug, version):
    # The pack's GitHub-ready repo file set — the exact files the
    # export ZIP holds and the publish upload sends, README override
    # included when the operator edited one.
    index = _rag_pack_index(slug)
    if index is None:
        raise ValueError("That pack is not in rag-packs/.")
    pdir = os.path.join(RAG_PACKS_DIR, slug)
    documents = []
    for fn in sorted(os.listdir(pdir)):
        if not fn.lower().endswith((".md", ".txt")):
            continue
        if fn.lower() == "readme.md":
            continue
        with open(os.path.join(pdir, fn)) as f:
            documents.append({"title": fn, "body": f.read()})
    if not documents:
        raise ValueError("That pack has no documents to export.")
    try:
        with open(os.path.join(pdir, "index.json")) as f:
            index_text = f.read()
    except Exception:
        index_text = ""
    files = build_rag_repo_files(
        index.get("name") or slug, index.get("description") or "",
        documents, version, index_text, icon=index.get("icon"),
    )
    override = rag_readme_override(slug)
    if override is not None:
        files = [(arc, override if arc == "README.md" else data) for arc, data in files]
    return files


def export_rag_pack(slug, version):
    # Builds the pack's GitHub-ready repo ZIP in rag-exports/ and
    # returns (zip file name, [repo file names]).
    files = rag_repo_files(slug, version)
    os.makedirs(RAG_EXPORTS_DIR, exist_ok=True)
    safe_version = re.sub(r"[^0-9A-Za-z.-]+", "-", version).strip("-.") or "0.1.0"
    zip_name = slug + "-" + safe_version + ".zip"
    with zipfile.ZipFile(os.path.join(RAG_EXPORTS_DIR, zip_name), "w", zipfile.ZIP_DEFLATED) as z:
        for arc, text in files:
            z.writestr(arc, text)
    return zip_name, [arc for arc, _text in files]


def build_chip_repo_files(chip, version):
    # The GitHub-ready repo layout for a chip — the format the
    # installer already reads, made explicit. Returns
    # [(arcname, text-or-bytes)] in repo order: README page,
    # navi-chip.json manifest, the chip's files at the root
    # (root.py is the entry; the rest are its importable modules),
    # the forged chip, .gitignore.
    slug = re.sub(r"[^a-z0-9]+", "-", chip["name"].lower()).strip("-") or "chip"
    chip_files = chip.get("files") or _valid_chip_files(None, chip.get("code") or "")
    # Issue #99: the export's version rides inside the forged
    # picture too, so the artifact states its own version wherever
    # it travels — the export's manifest and its picture agree.
    chip = dict(chip)
    chip["version"] = version
    # Build 3 (#101): the chip's declared icon art rides into the
    # forged picture and the export manifest alike; malformed art
    # refuses the export in plain words rather than shipping a
    # picture whose face silently drops it.
    if chip.get("icon"):
        if valid_icon_art(chip.get("icon")) is None:
            raise ValueError(ICON_ART_REFUSAL)
        chip["icon"] = valid_icon_art(chip.get("icon"))
    manifest = {
        "format": "navi-chip/1",
        "name": chip["name"],
        "description": chip.get("description") or "",
        "version": version,
        "entry": "root.py",
        "files": [f["path"] for f in chip_files],
        "output": chip.get("output") or "",
        "glyph": pick_chip_glyph(chip) or "",
        "icon": valid_icon_art(chip.get("icon")) or [],
        "generatedBy": "Apex — Chip Studio chip export",
    }
    lines = [
        "# " + chip["name"],
        "",
        chip.get("description") or "A chip for Apex.",
        "",
        "A **chip** for Apex — a small Python program Navi runs.",
        "The chip format is `navi-chip/1` (see `navi-chip.json`).",
        "",
        "## Files",
        "",
    ]
    for f in chip_files:
        note = " — the entry point Navi runs" if f["path"] == "root.py" else " — a module root.py can import"
        lines.append("- `" + f["path"] + "`" + note)
    lines += [
        "- `" + slug + ".png` — the chip as a forged chip (the whole",
        "  file set rides inside the image).",
        "",
        "## Install it in Navi",
        "",
        "1. Open Navi's **Chips** tab, then **Navi User Repo (NUR)**.",
        "2. **Search NUR** — every public repo tagged `navi-chips` is listed.",
        "3. **Install** this chip. Navi downloads every Python file at the",
        "   repo root at the latest release — `root.py` (or the entry named",
        "   in `navi-chip.json`) is the chip, the other files are its modules —",
        "   and forges them into a chip in the chips folder.",
        "",
        "## Publishing an update",
        "",
        "Edit the code, bump `version` in `navi-chip.json`, and press",
        "**Publish to GitHub** on the chip's card in Apex — Publish does",
        "everything: repo, files, topic, the version tag, and the published",
        "release with this picture attached (this export is `"
        + version + "`). Publishing by hand from an export ZIP works too —",
        "push the files and publish a release; the version tag is created",
        "with that publish. A newer release is what offers everyone who",
        "installed the chip an update. Keep the `navi-chips` topic on the",
        "repo — that is how Navi finds it.",
        "",
    ]
    gitignore = chr(10).join([".DS_Store", "Thumbs.db", "__pycache__/", ""])
    # An operator-edited README (kept inside the cartridge as
    # "readme") ships instead of the generated page, in the export
    # ZIP and the publish upload alike.
    readme_text = chip.get("readme") or chr(10).join(lines)
    files = [
        ("README.md", readme_text),
        ("navi-chip.json", json.dumps(manifest, indent=2) + chr(10)),
    ]
    for f in chip_files:
        files.append((f["path"], f["content"]))
    files.append((slug + ".png", build_cartridge_png(chip)))
    files.append((".gitignore", gitignore))
    return files


def load_local_chip(fname):
    path = os.path.join(CHIPS_DIR, os.path.basename(fname or ""))
    if not fname.endswith(".png") or not os.path.isfile(path):
        raise ValueError("That chip is not in the chips folder.")
    with open(path, "rb") as f:
        chip = parse_cartridge_png(f.read())
    if not chip:
        raise ValueError("That chip could not be read.")
    chip["file"] = os.path.basename(fname)
    return chip


def save_local_chip_readme(fname, readme):
    # Persists (or clears, when readme is None) the operator's
    # edited README inside the chip cartridge itself, so it travels
    # with the chip and survives re-export and re-publish.
    chip = load_local_chip(fname)
    if readme is None:
        chip.pop("readme", None)
    else:
        chip["readme"] = readme
    path = os.path.join(CHIPS_DIR, os.path.basename(fname))
    with open(path, "wb") as f:
        f.write(build_cartridge_png(chip))


def export_chip(fname, version):
    # Builds a local chip's GitHub-ready repo ZIP in
    # chips-exports/ and returns (zip file name, [repo file names]).
    chip = load_local_chip(fname)
    files = build_chip_repo_files(chip, version)
    os.makedirs(CHIPS_EXPORTS_DIR, exist_ok=True)
    slug = re.sub(r"[^a-z0-9]+", "-", chip["name"].lower()).strip("-") or "chip"
    safe_version = re.sub(r"[^0-9A-Za-z.-]+", "-", version).strip("-.") or "0.1.0"
    zip_name = slug + "-" + safe_version + ".zip"
    with zipfile.ZipFile(os.path.join(CHIPS_EXPORTS_DIR, zip_name), "w", zipfile.ZIP_DEFLATED) as z:
        for arc, data in files:
            z.writestr(arc, data)
    return zip_name, [arc for arc, _data in files]


def github_publish_files(repo_name, files, topic, description, release=None):
    # The publish loop, shared by RAG packs and chips — the WHOLE
    # chain in one action (0.1.9e): create the repo on the signed-in
    # account when it is not there (public — the catalogs only find
    # public repos), apply the topic with no prompt, upload the
    # exact file set the export builds, then create the tag and
    # PUBLISH the release (never a draft), named from the manifest
    # version, with the cartridge picture attached as the release
    # asset. The retired never-releases line is gone: Publish
    # does everything — repo, files, topic, tag, release, picture
    # attached. `release` carries {version, title, notes, assetName,
    # assetData}; when it is None the loop stops after the upload,
    # as before. A tag that already has a release is refused BEFORE
    # anything uploads — a release is never overwritten silently.
    # Returns a result dict; failures carry the real reason and
    # never fake a success (a failure after the upload says plainly
    # that the files landed and the release did not).
    auth = github_auth()
    if not auth:
        return {"ok": False, "error": "You are not signed in to GitHub. Sign in under Settings → GitHub & Backup, or use Export for GitHub and upload the ZIP yourself."}
    login, token = auth["login"], auth["token"]
    # Every call below goes through pub(): it carries the
    # current token. Renewal itself is the shared wrapper's
    # job since Build 11 (#197) — github_api / github_api_raw
    # renew the stored sign-in once on a 401 and retry once,
    # the behavior this loop first grew for itself (issue
    # #14); pub() only re-reads the stored token afterwards,
    # because a renewal rotates it mid-publish. A 401 that
    # survives the wrapper means the saved sign-in is dead,
    # and the operator hears that in plain words, never as a
    # raw HTTP string.
    state = {"token": token}

    def reauth_error():
        return {"ok": False, "error": "GitHub rejected the sign-in Navi has saved — the token has expired or been revoked, and Navi could not renew it. Sign out and back in under Settings → GitHub & Backup, then publish again. Nothing was uploaded."}

    def _resync_token():
        now = github_auth()
        if now and now.get("token"):
            state["token"] = now["token"]

    def pub(method, url, body=None):
        st, resp = github_api(method, url, state["token"], body)
        _resync_token()
        return st, resp

    def pub_raw(method, url, data, content_type):
        st, resp = github_api_raw(method, url, state["token"], data, content_type)
        _resync_token()
        return st, resp

    name = re.sub(r"[^A-Za-z0-9._-]+", "-", str(repo_name or "")).strip("-.")[:100]
    if not name:
        return {"ok": False, "error": "The repo name came out empty — give it a name before publishing."}
    if not login:
        st, me = pub("GET", "https://api.github.com/user")
        if st == 401:
            return reauth_error()
        if st == 200 and isinstance(me, dict):
            login = str(me.get("login") or "")
    if not login:
        return {"ok": False, "error": "GitHub did not say whose account this is — sign out and back in under Settings → GitHub & Backup, then try again."}
    base = "https://api.github.com/repos/" + login + "/" + name
    st, _repo = pub("GET", base)
    created = False
    if st == 401:
        return reauth_error()
    if st == 404:
        st2, resp = pub("POST", "https://api.github.com/user/repos", {
            "name": name, "description": description or "", "private": False,
        })
        if st2 == 401:
            return reauth_error()
        if st2 not in (200, 201):
            return {"ok": False, "error": "GitHub would not create the repo (HTTP " + str(st2) + "): " + github_err_text(resp) + " If the sign-in token cannot create repos, make a public repo named " + name + " on GitHub yourself and publish again — Navi will upload into it — or use Export for GitHub."}
        created = True
    elif st != 200:
        return {"ok": False, "error": "GitHub would not answer for that repo (HTTP " + str(st) + "): " + github_err_text(_repo)}
    # Release pre-check: a version that already has a release is
    # refused before a single file uploads — bump the version in
    # the manifest instead. (A leftover tag with NO release does
    # not block: creating the release below adopts it.)
    rel_tag = None
    if release and release.get("version"):
        rel_tag = "v" + str(release["version"]).strip().lstrip("v")
        stg, rel_existing = pub("GET", base + "/releases/tags/" + urllib.parse.quote(rel_tag))
        if stg == 401:
            return reauth_error()
        if stg == 200 and isinstance(rel_existing, dict):
            return {"ok": False, "error": rel_tag + " is already released on " + login + "/" + name + " — bump the version in the manifest and publish again. Nothing was uploaded and the existing release was not touched."}
    # Topic: merge with whatever is already there, never prompt.
    _st3, topics_resp = pub("GET", base + "/topics")
    if _st3 == 401:
        return reauth_error()
    names = topics_resp.get("names") if isinstance(topics_resp, dict) else []
    merged = sorted(set(list(names or []) + [topic]))
    _stt, _tresp = pub("PUT", base + "/topics", {"names": merged})
    if _stt == 401:
        return reauth_error()
    # Upload each file through the contents API (sha when updating).
    for arc, data in files:
        raw = data if isinstance(data, bytes) else str(data).encode("utf-8")
        body = {
            "message": ("Publish " if created else "Update ") + arc,
            "content": base64.b64encode(raw).decode("ascii"),
        }
        url = base + "/contents/" + urllib.parse.quote(arc)
        stx, existing = pub("GET", url)
        if stx == 401:
            return reauth_error()
        if stx == 200 and isinstance(existing, dict) and existing.get("sha"):
            body["sha"] = existing["sha"]
        st4, resp4 = pub("PUT", url, body)
        if st4 == 401:
            return reauth_error()
        if st4 not in (200, 201):
            return {"ok": False, "error": "Uploading " + arc + " failed (HTTP " + str(st4) + "): " + github_err_text(resp4)}
    result = {
        "ok": True, "error": None, "created": created,
        "repo": login + "/" + name,
        "repoUrl": "https://github.com/" + login + "/" + name,
        "files": [arc for arc, _d in files],
    }
    if rel_tag:
        notes = str(release.get("notes") or "")
        if len(notes) > 120000:
            notes = notes[:120000]
        st5, rel = pub("POST", base + "/releases", {
            "tag_name": rel_tag,
            "name": str(release.get("title") or rel_tag),
            "body": notes,
            "draft": False,
            "prerelease": False,
        })
        if st5 == 401:
            return {"ok": False, "error": "The files are uploaded to " + login + "/" + name + ", but GitHub rejected the sign-in when the release step ran and Navi could not renew it. Sign out and back in under Settings \u2192 GitHub & Backup, then publish again — the files are already there."}
        if st5 not in (200, 201) or not isinstance(rel, dict):
            return {"ok": False, "error": "The repo, topic, and files are on GitHub (" + login + "/" + name + "), but GitHub would not create the release (HTTP " + str(st5) + "): " + github_err_text(rel) + " If the sign-in token cannot create releases, publish the " + rel_tag + " release on GitHub yourself — the files and the picture are already uploaded."}
        rel_info = {
            "tag": rel_tag,
            "url": rel.get("html_url") or ("https://github.com/" + login + "/" + name + "/releases/tag/" + rel_tag),
            "asset": None,
        }
        asset_name = str(release.get("assetName") or "")
        asset_data = release.get("assetData")
        if asset_name and isinstance(asset_data, (bytes, bytearray)) and rel.get("id"):
            up_url = ("https://uploads.github.com/repos/" + login + "/" + name
                      + "/releases/" + str(rel["id"]) + "/assets?name="
                      + urllib.parse.quote(asset_name))
            st6, _asset = pub_raw("POST", up_url, bytes(asset_data), "application/octet-stream")
            if st6 == 401:
                return {"ok": False, "error": "The release " + rel_tag + " is published on " + login + "/" + name + ", but GitHub rejected the sign-in when the picture attach ran. Attach " + asset_name + " to the release on GitHub — it is already at the repo root too."}
            if st6 not in (200, 201):
                return {"ok": False, "error": "The release " + rel_tag + " is published on " + login + "/" + name + ", but attaching the picture failed (HTTP " + str(st6) + "): " + github_err_text(_asset) + " The picture is at the repo root; attach " + asset_name + " to the release on GitHub if you want it there too."}
            rel_info["asset"] = asset_name
        result["release"] = rel_info
    return result


def _rag_pack_index(slug):
    try:
        with open(os.path.join(RAG_PACKS_DIR, slug, "index.json")) as f:
            index = json.load(f)
        return index if isinstance(index, dict) else None
    except Exception:
        return None


# --- Knowledge cartridges (0.1.9e, issue #93) -----------------------
# A RAG pack can ride a picture the way a chip does: one PNG whose
# "navi-rag" tEXt chunk carries the pack's manifest and ALL of its
# document texts, zlib-compressed and base64'd. Installing from the
# picture validates it like a repo install and writes the pack with
# write_rag_pack — which builds the search index LOCALLY, at install
# time, from the carried texts. A cartridge never carries an index,
# and nothing at query time ever reads the picture: once installed,
# a pack is a pack. The ceiling is the finished picture's size: past
# it the answer is an honest refusal, never a silent truncation.
RAG_CARTRIDGE_MAX_BYTES = 8 * 1024 * 1024


def build_rag_cartridge_png(payload):
    # Build 2: the knowledge cartridge wears the same card-grade
    # face as a chip — the face builder reads the pack's own name,
    # description, version, and document count off the payload.
    # Build 3 (#101): a payload whose declared icon is damaged
    # refuses here, in plain words — forging a pack cartridge is a
    # forge path, and the face never silently drops the art. (The
    # PARSE side stays lenient: a damaged picture installs and
    # falls back.)
    if payload.get("icon"):
        if valid_icon_art(payload.get("icon")) is None:
            raise ValueError(ICON_ART_REFUSAL)
    face = _cartridge_face_png(payload, "navi-rag")
    blob = base64.b64encode(
        zlib.compress(json.dumps(payload).encode("utf-8"), 9)
    )
    text = b"navi-rag\x00" + blob
    ihdr_end = 8 + 12 + 13
    return face[:ihdr_end] + _png_chunk(b"tEXt", text) + face[ihdr_end:]


def parse_rag_cartridge_png(data):
    # (payload, error): the inverse of build_rag_cartridge_png,
    # with the Mesh card import's honesty rules — a picture with no
    # payload, a damaged one, and the wrong kind of picture each get
    # their own words.
    if not isinstance(data, (bytes, bytearray)) or len(data) < 8 or bytes(data[:8]) != b"\x89PNG\r\n\x1a\n":
        return None, "That file is not a PNG image — a knowledge cartridge travels as a .png picture."
    data = bytes(data)
    pos = 8
    blob = None
    saw_other = False
    while pos + 12 <= len(data):
        (length,) = struct.unpack(">I", data[pos:pos + 4])
        kind = data[pos + 4:pos + 8]
        if pos + 12 + length > len(data):
            return None, "This picture started life as a knowledge cartridge, but the data inside it is damaged — it no longer reads whole, so nothing was installed."
        chunk_data = data[pos + 8:pos + 8 + length]
        if kind == b"tEXt":
            nul = chunk_data.find(b"\x00")
            if nul > 0:
                if chunk_data[:nul] == b"navi-rag":
                    (crc_stored,) = struct.unpack(">I", data[pos + 8 + length:pos + 12 + length])
                    if binascii.crc32(kind + chunk_data) & 0xFFFFFFFF != crc_stored:
                        return None, "This picture started life as a knowledge cartridge, but the data inside it is damaged — it no longer reads whole, so nothing was installed."
                    blob = chunk_data[nul + 1:]
                else:
                    saw_other = True
        if kind == b"IEND":
            break
        pos += 8 + length + 4
    if blob is None:
        if saw_other:
            return None, "That picture carries a different kind of payload — it is not a knowledge cartridge. Knowledge cartridges are exported from RAG Studio."
        return None, "This image has no readable knowledge-cartridge data — it may have been recompressed by a chat app, which strips the data out of the picture while leaving the picture itself looking fine. Send the original file inside a .zip instead."
    try:
        payload = json.loads(zlib.decompress(base64.b64decode(blob)).decode("utf-8"))
    except Exception:
        return None, "This picture started life as a knowledge cartridge, but the data inside it is damaged — it no longer reads whole, so nothing was installed."
    if not isinstance(payload, dict) or payload.get("format") != "navi-rag/1":
        return None, "That picture's cartridge data is not in the shape a knowledge cartridge carries — nothing was installed."
    return payload, ""


def rag_cartridge_payload(slug):
    # The cartridge payload for an installed pack: its manifest and
    # every document's full text, read back from the pack folder.
    src = rag_pack_documents(slug)
    if src is None:
        return None
    index = _rag_pack_index(slug) or {}
    return {
        "format": "navi-rag/1",
        "name": src["name"],
        "description": src["description"],
        "version": "0.1.0",
        "documents": src["documents"],
        "generatedBy": "Apex — RAG Studio cartridge export",
        "chunkCountAtExport": index.get("chunkCount") or 0,
        # Build 3 (#101): the pack's declared art rides inside the
        # cartridge and draws its face, like a chip's icon.
        "icon": valid_icon_art(index.get("icon")) or [],
    }


def install_rag_cartridge(payload):
    # The one cartridge install path (Deck drops and the RAG tab's
    # picture install both land here). Validation mirrors a repo
    # install's caps; the pack is written by write_rag_pack, so its
    # index is built on this machine from the carried texts.
    name = str(payload.get("name") or "").strip()[:120]
    description = str(payload.get("description") or "")[:500]
    documents = []
    for raw in payload.get("documents") or []:
        if not isinstance(raw, dict):
            continue
        body = str(raw.get("body") or "")
        if not body.strip():
            continue
        if len(body) > RAG_MAX_DOC_CHARS:
            return None, "One document in this cartridge is longer than a pack document may be (" + str(RAG_MAX_DOC_CHARS) + " characters) — nothing was installed."
        documents.append({"title": str(raw.get("title") or "document")[:200], "body": body})
    if not name:
        return None, "This cartridge's pack has no name — nothing was installed."
    if not documents:
        return None, "This cartridge carries no documents — nothing was installed."
    if len(documents) > RAG_MAX_DOCS:
        return None, "This cartridge carries " + str(len(documents)) + " documents — a pack caps at " + str(RAG_MAX_DOCS) + ". Nothing was installed."
    rows = load_rag_installs()
    slug = chip_slug(name)
    taken = {r.get("local_dir") for r in rows if r.get("full_name") != "local/" + slug}
    taken |= {
        e for e in os.listdir(RAG_PACKS_DIR)
        if os.path.isdir(os.path.join(RAG_PACKS_DIR, e))
    } if os.path.isdir(RAG_PACKS_DIR) else set()
    if slug in taken or any(r.get("full_name") == "local/" + slug for r in rows):
        return None, "A pack named " + name + " is already on this machine — remove it first, or rename the cartridge's pack."
    write_rag_pack(slug, name, description, documents,
                   icon=valid_icon_art(payload.get("icon")))
    rows.append({
        "full_name": "local/" + slug,
        "name": name,
        "html_url": "",
        "installed_tag": None,
        "installed_ref": "knowledge cartridge (.png)",
        "installed_pushed_at": None,
        "installed_at": _now_iso(),
        "local_dir": slug,
        "active": True,
        "doc_count": len(documents),
    })
    save_rag_installs(rows)
    return slug, ""


def local_rag_packs():
    # Every folder under rag-packs/ holding a readable index is a
    # pack on this machine; the registry supplies its source and
    # active flag. A folder with no record is a local pack, active
    # by default — the same rule as a chip dropped into chips/.
    by_dir = {}
    for r in load_rag_installs():
        if r.get("local_dir"):
            by_dir[r["local_dir"]] = r
    packs = []
    try:
        entries = sorted(os.listdir(RAG_PACKS_DIR))
    except Exception:
        entries = []
    for entry in entries:
        if not os.path.isdir(os.path.join(RAG_PACKS_DIR, entry)):
            continue
        index = _rag_pack_index(entry)
        if index is None:
            continue
        rec = by_dir.get(entry)
        full = rec.get("full_name") if rec else "local/" + entry
        packs.append({
            "slug": entry,
            "name": index.get("name") or entry,
            "description": index.get("description") or "",
            "fullName": full,
            "source": "github" if rec and not str(full).startswith("local/") else "local",
            "installedRef": _display_ref(rec.get("installed_ref")) if rec else None,
            "locallyEdited": bool(rec.get("locally_edited")) if rec else False,
            "active": bool(rec.get("active", True)) if rec else True,
            "docCount": len(index.get("documents") or []),
            "chunkCount": index.get("chunkCount") or 0,
            "documents": [d.get("title") for d in (index.get("documents") or [])],
        })
    return packs


def rag_pack_documents(slug):
    # A pack's documents with their bodies, for reopening the pack in
    # RAG Studio: the index holds the titles, and each body is the
    # document file write_rag_pack laid down under the same flattened
    # name (PDF text already lives as .txt under the PDF's stem).
    index = _rag_pack_index(slug)
    if index is None:
        return None
    pdir = os.path.join(RAG_PACKS_DIR, slug)
    used = set()
    documents = []
    for d in index.get("documents") or []:
        title = d.get("title") or "document"
        flat = title.replace("/", "-")
        safe = re.sub(r"[^A-Za-z0-9._-]+", "-", flat).strip("-.") or "document"
        if safe.lower().endswith(".pdf"):
            safe = safe[:-4] + ".txt"
        elif not safe.lower().endswith((".md", ".txt")):
            safe += ".md"
        stem, dot, ext = safe.rpartition(".")
        cand = safe
        n = 2
        while cand in used:
            cand = (stem + "-" + str(n) + "." + ext) if dot else (safe + "-" + str(n))
            n += 1
        used.add(cand)
        body = ""
        try:
            with open(os.path.join(pdir, cand)) as f:
                body = f.read()
        except Exception:
            body = ""
        documents.append({"title": title, "body": body})
    return {
        "name": index.get("name") or slug,
        "description": index.get("description") or "",
        "documents": documents,
    }


# ------------------------------- The Vault -------------------------------
# Build .72, issue #54: a shared desk of plain markdown notes at
# config/vault/ inside the install — real .md files in subfolders,
# YAML frontmatter preserved, [[wikilinks]] and #tags as plain text,
# so the operator can open the same folder in Obsidian and
# everything works. Every path is jailed to the vault root; writes
# land atomically (temp file + rename); the seed runs only when the
# folder does not exist yet and never touches an existing file.

VAULT_DIR = os.path.join(ROOT, "config", "vault")


def _vault_seed_files():
    today = time.strftime("%Y-%m-%d", time.localtime())
    welcome = "\n".join([
        "---", "tags: [vault, guide]", "---", "",
        "# Welcome to the Vault", "",
        "This folder is a shared desk: plain markdown files you can open here, in Obsidian, or in any text editor — same files, no conversion, nothing locked in.", "",
        "## The two areas", "",
        "- **Navi's Desk/** — her side of the desk. When a conversation produces something worth keeping — a decision, a finding, something you asked her to remember — she files a dated note here and tells you she did.",
        "- **Operator/** — your side. She never edits your notes silently: she writes here only when you ask her to in chat, and she says what she filed.",
        "- **Daily/** — one note per day, from the Daily Note button.",
        "- **Templates/** — files here are offered as starting templates when you make a new note.", "",
        "## Conventions", "",
        "- Link notes with [[wikilinks]] — click one in Preview to jump (or to create the note).",
        "- Tag with #tags anywhere in the text.",
        "- YAML frontmatter at the top of a note stays exactly as written; the preview never mangles it.", "",
        "The vault folder rides the installer's keep-list, so updates and fresh installs preserve it. See [[Start here]] on your side of the desk, and [[First note]] on hers.", "",
    ])
    first = "\n".join([
        "---", "filed: " + today, "---", "",
        "# First note", "",
        "Filed " + today + " — my first note on my side of the desk.", "",
        "This is where I'll keep what a conversation produces that's worth more than the conversation: decisions we made and why, findings from the work, things you asked me to remember. Dated, first person, in my own words — and when one of these notes informs an answer I give you, I'll name it as a source, the way I name a RAG pack.", "",
        "Your notes live in [[Start here]]'s folder, and I won't touch them unless you ask. The desk is shared; the sides are respected.", "",
        "#navi #desk", "",
    ])
    start = "\n".join([
        "# Start here", "",
        "Your side of the desk. Make a note with **New note**, drag notes between folders in the tree, or drop markdown files from your computer onto the vault to import them.", "",
        "Ask Navi in chat to file something here and she will — out loud, never silently. Her own notes live across the desk in [[First note]].", "",
    ])
    template = "\n".join([
        "---", "tags: []", "---", "",
        "# Untitled", "",
        "## Notes", "", "- ", "",
        "## Links", "", "- [[Welcome]]", "",
    ])
    return [
        ("Welcome.md", welcome),
        ("Navi's Desk/First note.md", first),
        ("Operator/Start here.md", start),
        ("Templates/Note template.md", template),
    ]


def vault_join(rel):
    # The jail: normalize a vault-relative path, or return None for
    # anything that escapes the vault root (.., absolute, empty).
    rel = str(rel or "").replace(chr(92), "/").strip()
    if not rel or rel.startswith("/"):
        return None
    parts = []
    for seg in rel.split("/"):
        if seg in ("", "."):
            continue
        if seg == "..":
            return None
        parts.append(seg)
    if not parts:
        return None
    root = os.path.abspath(VAULT_DIR)
    full = os.path.abspath(os.path.join(root, *parts))
    if full != root and not full.startswith(root + os.sep):
        return None
    return "/".join(parts)


def vault_fs_path(rel):
    return os.path.join(VAULT_DIR, *rel.split("/"))


def vault_write(rel, content):
    full = vault_fs_path(rel)
    parent = os.path.dirname(full)
    if parent:
        os.makedirs(parent, exist_ok=True)
    tmp = full + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        f.write(str(content))
    os.replace(tmp, full)


VAULT_RAW_IMAGE_TYPES = {
    ".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
    ".gif": "image/gif", ".webp": "image/webp",
}


def vault_raw_payload(rel):
    # v0.5.3e (#265): one Vault file's bytes + content type for
    # the /api/vault/raw door, or (None, None). The caller has
    # already jailed rel through vault_join. Raster images are
    # images; anything else is opaque bytes (the door adds a
    # sandbox policy and nosniff — a filed file is shown,
    # never executed in the dashboard's origin).
    full = vault_fs_path(rel)
    if not os.path.isfile(full):
        return None, None
    try:
        with open(full, "rb") as f:
            data = f.read()
    except Exception:
        return None, None
    ext = os.path.splitext(rel)[1].lower()
    return data, VAULT_RAW_IMAGE_TYPES.get(ext, "application/octet-stream")


def vault_delete_note(rel):
    # One note out of the desk (Build 3, #177: factored out of the
    # /api/vault/delete handler so the deletion's tombstone lives
    # in exactly one place). Returns True when a file went.
    full = vault_fs_path(rel)
    if not os.path.isfile(full):
        return False
    os.remove(full)
    sync_record_tombstone("vault", rel)
    return True


def vault_delete_folder(rel):
    # A folder and everything in it (the handler's walk, kept).
    # Every note inside is tombstoned by its vault path; when the
    # folder IS a case folder (cases/<slug>), the case itself is
    # tombstoned too, so deleting a case from the Vault's tree
    # cannot resurrect through sync either. Returns the number of
    # markdown notes removed, exactly as the handler always has.
    full = vault_fs_path(rel)
    count = 0
    if os.path.isdir(full):
        notes = []
        for base, dirs, files in os.walk(full):
            for name in files:
                if name.lower().endswith(".md"):
                    notes.append(os.path.relpath(os.path.join(base, name), VAULT_DIR).replace(os.sep, "/"))
                    count += 1
        shutil.rmtree(full)
        for note_rel in notes:
            sync_record_tombstone("vault", note_rel)
        parts = rel.split("/")
        if len(parts) == 2 and parts[0] == "cases":
            sync_record_tombstone("cases", parts[1])
    return count


def vault_ensure_seed():
    # First run only: if the folder already exists, the operator's
    # desk is theirs — seeding never writes into it again.
    if os.path.isdir(VAULT_DIR):
        return
    os.makedirs(VAULT_DIR, exist_ok=True)
    for rel, content in _vault_seed_files():
        if not os.path.exists(vault_fs_path(rel)):
            vault_write(rel, content)


def _vault_links(content):
    out = []
    for m in re.finditer(r"\[\[([^\]|]+)(?:\|[^\]]+)?\]\]", content or ""):
        t = m.group(1).strip()
        if t and t not in out:
            out.append(t)
    return out


def _vault_tags(content):
    out = []
    for m in re.finditer(r"(?m)(?:^|\s)#([A-Za-z][A-Za-z0-9_-]*)", content or ""):
        t = m.group(1)
        if t and t not in out:
            out.append(t)
    return out


def _vault_is_case_fragment(rel):
    # v0.6.4e (#320): a case's WORKING folder (cases/<slug>/…)
    # is the case manager's archive, not Vault reading matter —
    # its fragments (manifest, findings, registers, timeline,
    # custody) never surface as notes. The case itself reads
    # as ONE formed file on its type shelf (see
    # case_formed_rel). The legacy formed document
    # (cases/<slug>.md, two segments) is not a fragment.
    parts = rel.split("/")
    return len(parts) >= 3 and parts[0] == "cases"


def vault_notes_raw():
    vault_ensure_seed()
    notes = []
    if os.path.isdir(VAULT_DIR):
        for base, dirs, files in os.walk(VAULT_DIR):
            dirs.sort()
            for name in sorted(files):
                if not name.lower().endswith(".md"):
                    continue
                full = os.path.join(base, name)
                rel = os.path.relpath(full, VAULT_DIR).replace(os.sep, "/")
                if _vault_is_case_fragment(rel):
                    continue
                try:
                    with open(full, encoding="utf-8") as f:
                        content = f.read()
                except Exception:
                    continue
                notes.append({
                    "path": rel,
                    "content": content,
                    "updatedAt": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(os.path.getmtime(full))),
                })
    notes.sort(key=lambda n: n["path"])
    return notes


def vault_list_payload():
    notes = vault_notes_raw()
    folders = set()
    if os.path.isdir(VAULT_DIR):
        for base, dirs, files in os.walk(VAULT_DIR):
            for d in dirs:
                rel = os.path.relpath(os.path.join(base, d), VAULT_DIR).replace(os.sep, "/")
                # v0.6.4e (#320): the working archive's folders
                # are not shelf structure — the listing's
                # folders are the shelves and the free notes'
                # homes, never cases/<slug> machinery.
                if rel == "cases" or rel.startswith("cases/"):
                    continue
                folders.add(rel)
    # v0.6.4e (#320): formed documents are marked by the
    # bridge that formed them — the reader's element strip
    # (and any other consumer) keys off the mark, never off a
    # path shape that a rename could silently break.
    try:
        _formed_cases = {}
        for _slug in _case_all_slugs():
            _formed_cases[case_formed_rel(_slug)] = _slug
        _formed_dossiers = {}
        _dstore = _dossier_store()
        for _d in _dstore.get("dossiers") or []:
            _formed_dossiers[dossier_formed_rel(_d, _dstore)] = _d["id"]
    except Exception:
        _formed_cases, _formed_dossiers = {}, {}
    for n in notes:
        parts = n["path"].split("/")
        title = parts[-1][:-3] if parts[-1].lower().endswith(".md") else parts[-1]
        n["title"] = title
        n["outgoing"] = _vault_links(n["content"])
        n["tags"] = _vault_tags(n["content"])
        if n["path"] in _formed_cases:
            n["formedCase"] = _formed_cases[n["path"]]
        if n["path"] in _formed_dossiers:
            n["formedDossier"] = _formed_dossiers[n["path"]]
    for n in notes:
        n["backlinks"] = [
            o["path"] for o in notes
            if o["path"] != n["path"]
            and any(t.lower() == n["title"].lower() for t in o["outgoing"])
        ]
    # v0.6.6e (#328): the Cases section stands at the BOTTOM
    # of the Vault — after every other folder, never sorted
    # into the middle of the operator's own shelves. The
    # payload carries the order; the view honors it.
    ordered = sorted(f for f in folders if f != "Cases" and not f.startswith("Cases/"))
    ordered += sorted(f for f in folders if f == "Cases" or f.startswith("Cases/"))
    return {"notes": notes, "folders": ordered}


def vault_search(query, limit=2):
    words = set(w for w in re.findall(r"[a-z0-9][a-z0-9_-]{3,}", (query or "").lower()))
    if not words:
        return []
    scored = []
    for n in vault_notes_raw():
        title = n["path"].split("/")[-1].lower()
        tags = [t.lower() for t in _vault_tags(n["content"])]
        body = n["content"].lower()
        score = 0
        for w in words:
            if w in title:
                score += 3
            if w in tags:
                score += 2
            if w in body:
                score += 1
        if score > 0:
            prose = re.sub(r"^---\n.*?\n---\n", "", n["content"], flags=re.S)
            lines = [ln.strip() for ln in prose.split("\n") if ln.strip() and not ln.strip().startswith("#")]
            scored.append({"path": n["path"], "score": score, "excerpt": " ".join(lines)[:400]})
    scored.sort(key=lambda h: -h["score"])
    return scored[:limit]


def vault_file_from_chat(text, history=None):
    # Her note-taking from chat (issue #54): a filing request lands
    # on her side of the desk — Navi's Desk/, dated, first person —
    # unless the operator names their own side. The note is ABOUT
    # something: the thing just discussed. A bare request ("make a
    # note about that up there") names no subject, so the recent
    # conversation rides along (build .73, field report: the first
    # version filed an empty note under a promising name): the note
    # says what it is about and quotes the exchange it came from —
    # above all the latest terminal output. Returns (path, err).
    raw = (text or "").strip()
    to_operator = bool(re.search(r"\b(?:in|to|under)\s+(?:my|the\s+operator|operator)\b", raw, re.I))
    body = raw.split(":", 1)[1].strip() if ":" in raw else raw
    body = re.sub(r"^(?:please\s+)?(?:file|write|save|make|take)\s+(?:a\s+|this\s+)?note\b", "", body, flags=re.I).strip()
    body = re.sub(r"\bnote\s+(?:this|that)\s+down\b", "", body, flags=re.I).strip() or raw
    # The subject as the operator phrased it, with the politeness and
    # the filing verbs stripped: "Can you make a note about X" -> "X".
    subject = body
    subject = re.sub(r"^(?:please\s+)?(?:can|could|would)\s+you\s+", "", subject, flags=re.I).strip()
    subject = re.sub(r"^(?:please\s+)?(?:file|write|save|make|take)\s+(?:a\s+|this\s+)?note\s+(?:about|on|of)\s+", "", subject, flags=re.I).strip()
    subject = re.sub(r"^(?:please\s+)?(?:file|write|save|make|take)\s+(?:a\s+|this\s+)?note\b\s*", "", subject, flags=re.I).strip(" .?:")
    # Recent conversation, newest first, skipping the request itself.
    ctx = []
    for item in reversed(history or []):
        if not isinstance(item, dict):
            continue
        c = str(item.get("content") or "").strip()
        if not c or c == raw:
            continue
        ctx.append(c)
        if len(ctx) >= 3:
            break
    term = ""
    for c in ctx:
        if "terminal" in c.lower() or c.startswith("$ "):
            term = c
            break
    term_cmd = ""
    if term:
        for ln in term.split("\n"):
            ln = ln.strip()
            if ln.startswith("$ "):
                term_cmd = ln[2:].strip()
                break
    vague = (not subject) or len(subject.split()) < 2 or bool(re.match(r"(?i)^(that|this|it|the above|up there)\b", subject))
    # Her titles name the SUBJECT, never the operator's ask (field
    # report: a filing named "Can you make a note" after his literal
    # words). A vague ask over a terminal result titles the note for
    # what the terminal showed — for a listing, the place listed and
    # the first thing in it.
    if not vague:
        title_src = subject
    elif term_cmd:
        title_src = ""
        m_ls = re.match(r"ls\s+(?:-\S+\s+)*(\S+)", term_cmd)
        if m_ls:
            seg = m_ls.group(1).rstrip("/").split("/")[-1] or "root"
            first_entry = ""
            seen_cmd = False
            for ln in term.split("\n"):
                s_ln = ln.strip()
                if s_ln.startswith("$ "):
                    seen_cmd = True
                    continue
                if seen_cmd and s_ln:
                    tok = s_ln.split()[0]
                    if not tok.endswith(":"):
                        first_entry = tok
                        break
            title_src = seg + " listing - " + first_entry if first_entry else seg + " listing"
        if not title_src:
            title_src = "Terminal - " + term_cmd
    elif ctx:
        title_src = ctx[-1]
    else:
        title_src = subject or body
    if len(title_src) > 80:
        title_src = " ".join(title_src.split()[:6])
    title = " ".join(title_src.replace("#", "").split()).strip(" .:") or "Untitled note"
    title = title.replace("/", "-").replace(chr(92), "-")
    today = time.strftime("%Y-%m-%d", time.localtime())
    folder = "Operator" if to_operator else "Navi's Desk"
    rel = folder + "/" + today + " " + title + ".md"
    n = 2
    while os.path.exists(vault_fs_path(rel)):
        rel = folder + "/" + today + " " + title + " " + str(n) + ".md"
        n += 1
    if to_operator:
        content = "# " + title + "\n\n" + body + "\n"
    else:
        lines = [
            "---", "filed: " + today, "---", "",
            "# " + title, "",
            "Alex asked me to keep a note of this, " + today + " — in his words: “" + raw + "”", "",
            "## What it's about", "",
        ]
        if not vague:
            lines += [subject[0].upper() + subject[1:] if subject else subject, ""]
        elif term_cmd:
            lines += ["The terminal output from just before he asked — the “" + term_cmd + "” run, kept below so the note stands on its own.", ""]
        elif ctx:
            lines += ["The exchange just before he asked, kept below so the note stands on its own.", ""]
        else:
            lines += ["He asked with nothing else on the table — the request itself is the record, and this line is me saying so instead of filing an empty page.", ""]
        if ctx:
            lines += ["## From the conversation", ""]
            for c in reversed(ctx):
                excerpt = c if len(c) <= 900 else c[:900] + "\n... [trimmed]"
                for ln in excerpt.split("\n"):
                    lines.append("    " + ln)
                lines.append("")
        lines += ["— filed by Navi from chat, " + today + ".", ""]
        content = "\n".join(lines)
    vault_write(rel, content)
    return rel, None


def rag_retrieve(question, limit=2):
    # Only active packs feed her answers. Only the question is
    # embedded here — every chunk vector was computed when its pack
    # was installed or forged.
    qv = _rag_embed(question)
    if not any(qv):
        return []
    by_dir = {}
    for r in load_rag_installs():
        if r.get("local_dir"):
            by_dir[r["local_dir"]] = r
    hits = []
    try:
        entries = sorted(os.listdir(RAG_PACKS_DIR))
    except Exception:
        entries = []
    for entry in entries:
        rec = by_dir.get(entry)
        if rec is not None and not rec.get("active", True):
            continue
        index = _rag_pack_index(entry)
        if index is None:
            continue
        pack_name = index.get("name") or entry
        for doc in index.get("documents") or []:
            for chunk in doc.get("chunks") or []:
                vec = chunk.get("vector") or []
                score = sum(a * b for a, b in zip(qv, vec))
                if score >= RAG_MIN_SCORE:
                    hits.append({
                        "pack": pack_name,
                        "document": doc.get("title") or "",
                        "text": chunk.get("text") or "",
                        "score": score,
                    })
    hits.sort(key=lambda h: -h["score"])
    return hits[:limit]
def strip_citation_footer(text):
    # A citation footer — a trailing "— from <pack> · <document>"
    # line — is presentation, not prose: sources ride the chat
    # payload's "rag" field and the dashboard draws them as pills
    # under the message. They never live in reply text, so her
    # voice never reads them aloud and they never enter the history
    # that would teach her to imitate them. This strips a trailing
    # footer in that shape wherever one survives: a model's own
    # imitation, or a thread an older build stored.
    lines = str(text or "").rstrip().split("\n")
    while lines and lines[-1].strip().startswith("— from "):
        lines.pop()
        while lines and not lines[-1].strip():
            lines.pop()
    return "\n".join(lines).rstrip()


def fetch_pack_documents(repo):
    # Downloads the pack itself from its repo and returns its
    # documents ([{title, body}] — title is the repo-relative path,
    # which is what an answer cites). A navi-rag repo carries its
    # documents as .md/.txt/.pdf files anywhere in the tree; they
    # are fetched at the installed ref (the latest release's tag
    # when the repo cuts releases, otherwise the default branch),
    # and a PDF's body is its extracted text. Raises ValueError
    # with the reason when the repo holds no documents, so an
    # install never records a pack that is not actually here.
    full = repo["fullName"]
    latest = repo["latestRelease"]
    ref = latest["tag"] if latest else (repo["defaultBranch"] or "main")
    tree = github_json(
        "https://api.github.com/repos/" + full + "/git/trees/"
        + urllib.parse.quote(ref, safe="") + "?recursive=1"
    )
    paths = []
    for item in tree.get("tree") or []:
        if item.get("type") != "blob":
            continue
        path = str(item.get("path") or "")
        low = path.lower()
        # A root README.md is the pack's GitHub page, not a
        # document: it describes the pack and is never indexed.
        # (A README inside a subfolder is an ordinary document.)
        if low == "readme.md":
            continue
        if low.endswith(".md") or low.endswith(".txt") or low.endswith(".pdf"):
            paths.append(path)
    if not paths:
        raise ValueError(
            "the repo has no .md, .txt or .pdf documents at " + ref
            + " — a navi-rag repo carries its documents as Markdown, text, or PDF files"
            + " (a root README.md describes the pack and doesn't count)."
        )
    if len(paths) > RAG_MAX_DOCS:
        raise ValueError(
            "the repo holds " + str(len(paths)) + " documents — a pack caps at "
            + str(RAG_MAX_DOCS) + " so an install stays lean on a small machine."
        )
    documents = []
    for path in sorted(paths):
        url = (
            "https://raw.githubusercontent.com/" + full + "/"
            + urllib.parse.quote(ref, safe="") + "/" + urllib.parse.quote(path)
        )
        raw = github_download(url)
        if path.lower().endswith(".pdf"):
            body = pdf_extract_text(raw)
            if not body:
                raise ValueError(
                    path + " has no extractable text — a scanned or "
                    "image-only PDF can't be indexed."
                )
        else:
            body = raw.decode("utf-8", "replace")
        documents.append({"title": path, "body": body[:RAG_MAX_DOC_CHARS]})
    return documents


def rag_install_record(repo, slug, doc_count):
    record = chip_install_record(repo)
    record["local_dir"] = slug
    record["active"] = True
    record["doc_count"] = doc_count
    return record


def rag_pack_slug(repo, rows):
    # The folder a GitHub install writes into rag-packs/, named for
    # the repo; when another pack already owns that slug, the
    # owner's name joins it instead of overwriting their pack.
    slug = chip_slug(repo["name"])
    taken = {
        r.get("local_dir") for r in rows
        if r.get("full_name") != repo["fullName"]
    }
    if slug in taken:
        slug = chip_slug(repo["owner"] + "-" + repo["name"])
    return slug


def rag_catalog_view():
    # The chips catalog view, for packs: the GitHub topic search
    # merged with the install registry, plus forged local packs.
    error = None
    warning = None
    repos = []
    try:
        # Build 2 (#173): same discipline as the chips catalog —
        # Runtime credential, five-minute cache, one search in
        # flight, failures classified and named.
        data = github_search_repositories("topic:navi-rag")
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
                + " of the packs — the list may be incomplete. "
                + github_error_text(last_err, authed=bool(github_read_token()))
            )
    except Exception as e:
        error = "GitHub search failed. " + github_error_text(e, authed=bool(github_read_token()))
    installs = {row.get("full_name"): row for row in load_rag_installs()}
    # Issue #22, the pack half — the case from Alex's screenshot:
    # a pack forged in RAG Studio (its local/ record, folder named
    # for the pack) must read as installed on its repo's catalog
    # card, not NOT INSTALLED with a live Install above the local
    # card proving otherwise. Same slug rule as chips: the repo
    # name is matched against the pack folders on this machine
    # and the install records, local/ ones included. An exact
    # full-name match keeps the update math; a slug match claims
    # only "installed" and borrows its state from the local pack.
    installs_ci = {}
    rec_by_slug = {}
    for row in load_rag_installs():
        fn = row.get("full_name") or ""
        if fn:
            installs_ci.setdefault(fn.lower(), row)
        s = fn.split("/")[-1].lower()
        if s and s not in rec_by_slug:
            rec_by_slug[s] = row
    local_packs = {}
    for lp in local_rag_packs():
        local_packs.setdefault(str(lp.get("slug") or "").lower(), lp)
    packs = []
    for repo in repos:
        inst = installs.get(repo["fullName"]) or installs_ci.get(repo["fullName"].lower())
        slug_lp = None
        slug_rec = None
        if not inst:
            s = repo["fullName"].split("/")[-1].lower()
            slug_lp = local_packs.get(s)
            slug_rec = rec_by_slug.get(s)
        slug_matched = bool(not inst and (slug_lp is not None or slug_rec is not None))
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
            local_dir = inst.get("local_dir")
            active = bool(inst.get("active", True))
            doc_count = inst.get("doc_count")
            edited = bool(inst.get("locally_edited"))
            tag = inst.get("installed_tag")
            at = inst.get("installed_at")
        elif slug_matched:
            ref = slug_lp.get("installedRef") if slug_lp else (
                _display_ref(slug_rec.get("installed_ref"))
                if slug_rec and str(slug_rec.get("full_name") or "").startswith("local/")
                else None
            )
            local_dir = slug_lp.get("slug") if slug_lp else (slug_rec.get("local_dir") if slug_rec else None)
            active = slug_lp.get("active") if slug_lp else (bool(slug_rec.get("active", True)) if slug_rec else None)
            doc_count = slug_lp.get("docCount") if slug_lp else (slug_rec.get("doc_count") if slug_rec else None)
            edited = slug_lp.get("locallyEdited") if slug_lp else (bool(slug_rec.get("locally_edited")) if slug_rec else False)
            tag = slug_rec.get("installed_tag") if slug_rec else None
            at = slug_rec.get("installed_at") if slug_rec else None
        else:
            ref = None
            local_dir = None
            active = None
            doc_count = None
            edited = False
            tag = None
            at = None
        entry = dict(repo)
        entry["inCatalog"] = True
        entry["installed"] = bool(inst) or slug_matched
        entry["slugMatched"] = slug_matched
        entry["installedTag"] = tag
        entry["installedRef"] = ref
        entry["installedAt"] = at
        entry["updateAvailable"] = update
        entry["localDir"] = local_dir
        entry["active"] = active
        entry["docCount"] = doc_count
        entry["locallyEdited"] = edited
        packs.append(entry)
    seen = {p["fullName"] for p in packs}
    for full, inst in installs.items():
        if full in seen:
            continue
        packs.append({
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
            "localDir": inst.get("local_dir"),
            "active": bool(inst.get("active", True)),
            "docCount": inst.get("doc_count"),
            "locallyEdited": bool(inst.get("locally_edited")),
        })
    view = {"fetchedAt": _now_iso(), "packs": packs}
    if error:
        view["error"] = error
    if warning:
        view["warning"] = warning
    return view


# ------------------------------- Voice -------------------------------
# Her voice, fully local. The bridge stays stdlib-only: the speech
# engines (faster-whisper for hearing, Kokoro or Piper for speaking)
# are optional dependencies, imported lazily the first time a voice
# endpoint is used. When they are not importable, the endpoints say
# so, with the step that installs them — nothing fakes audio.
# Settings live in navi.config.json's "voice" section; models land
# in voice/models/ on first use; the live call is voice/state.json
# and finished calls are archived to voice/calls.json.

VOICE_DIR = os.path.join(ROOT, "voice")
VOICE_MODELS_DIR = os.path.join(VOICE_DIR, "models")
VOICE_STATE_PATH = os.path.join(VOICE_DIR, "state.json")
VOICE_CALLS_PATH = os.path.join(VOICE_DIR, "calls.json")


def _adopt_voice_venv():
    # The installer lays the speech engines down in an isolated
    # venv at voice/venv, and the launchers run this bridge with
    # that venv's Python — but a bridge started any other way (by
    # hand, by an older launcher, or one already running from
    # before the voice step) runs under the system Python and
    # would report her voice missing while the engines sit right
    # beside it. Adopt the venv's site-packages into this
    # interpreter's module path, so the engines the installer laid
    # down are the engines this bridge finds, whichever Python
    # started it. The installer builds the venv from the system
    # python3, so the packages are built for this Python family
    # and import identically. Returns the adopted path, or None.
    venv = os.path.join(VOICE_DIR, "venv")
    candidates = []
    lib = os.path.join(venv, "lib")
    if os.path.isdir(lib):
        try:
            for entry in sorted(os.listdir(lib)):
                candidates.append(os.path.join(lib, entry, "site-packages"))
        except Exception:
            pass
    candidates.append(os.path.join(venv, "Lib", "site-packages"))
    for path in candidates:
        if os.path.isdir(path):
            if path not in sys.path:
                sys.path.insert(0, path)
            return path
    return None


VOICE_VENV_SITE_PACKAGES = _adopt_voice_venv()
VOICE_INSTALL_HINT = (
    "Install her voice: re-run the installer and say yes at the voice "
    "step, or run: python3 -m pip install -r requirements-voice.txt"
)

VOICE_DEFAULTS = {
    "enabled": True,
    "stt_model": "small.en",
    "tts_engine": "kokoro",
    "tts_voice": "bf_isabella",
    "rate": 1.0,
    "pitch": 1.0,
}


class VoiceUnavailable(Exception):
    pass


def voice_config(cfg):
    raw = cfg.get("voice") or {}
    stt = raw.get("stt") or {}
    tts = raw.get("tts") or {}
    out = dict(VOICE_DEFAULTS)
    if isinstance(raw.get("enabled"), bool):
        out["enabled"] = raw["enabled"]
    if stt.get("model"):
        out["stt_model"] = str(stt["model"])
    if tts.get("engine") in ("kokoro", "piper"):
        out["tts_engine"] = tts["engine"]
    if tts.get("voice"):
        out["tts_voice"] = str(tts["voice"])
    for key in ("rate", "pitch"):
        try:
            val = float(tts.get(key))
            if 0.5 <= val <= 2.0:
                out[key] = val
        except (TypeError, ValueError):
            pass
    return out


def save_voice_config(patch):
    # Voice settings persist into navi.config.json, beside the rest
    # of the app config — the dashboard's Voice tab writes them here.
    try:
        with open(CONFIG_PATH) as f:
            cfg = json.load(f)
    except Exception:
        cfg = {}
    cur = voice_config(cfg)
    for key, val in patch.items():
        if key in VOICE_DEFAULTS:
            cur[key] = val
    cfg["voice"] = {
        "enabled": bool(cur["enabled"]),
        "stt": {"engine": "faster-whisper", "model": cur["stt_model"]},
        "tts": {
            "engine": cur["tts_engine"],
            "voice": cur["tts_voice"],
            "rate": cur["rate"],
            "pitch": cur["pitch"],
        },
    }
    os.makedirs(os.path.dirname(CONFIG_PATH), exist_ok=True)
    with open(CONFIG_PATH, "w") as f:
        json.dump(cfg, f, indent=2)
    return cur


def _module_available(name):
    try:
        return importlib.util.find_spec(name) is not None
    except Exception:
        return False


_engine_probes = {}


def _probe_engine(pkg, symbol):
    # The one availability truth for a speech engine: the exact import
    # the synthesis/transcription paths perform (from kokoro import
    # KPipeline, from faster_whisper import WhisperModel), attempted
    # once in this interpreter and cached. find_spec alone is not
    # proof — it locates a package's directory even when that
    # package's own imports fail under this Python, which is what
    # happens when the bridge runs under a different interpreter than
    # the voice venv's and merely adopted its site-packages. That
    # split is how every status surface could call Kokoro installed
    # while a speak attempt called it missing. voice_status reports
    # this probe and the loaders fail with the same distinction, so
    # the pipeline strip, the status column, and the banner can never
    # disagree again. Returns (available, exception-or-None).
    key = pkg + ":" + symbol
    if key not in _engine_probes:
        try:
            mod = importlib.import_module(pkg)
            if symbol:
                getattr(mod, symbol)
            _engine_probes[key] = (True, None)
        except Exception as e:
            _engine_probes[key] = (False, e)
    return _engine_probes[key]


def _engine_detail(pkg, exc):
    # Why an engine is unavailable, in words that name the failing
    # piece: a genuinely absent package gets the install step; a
    # package that is present but fails to load says so and carries
    # the real import error instead of hiding behind "not installed".
    if exc is None:
        return None
    if not _module_available(pkg):
        return ("The " + pkg + " package is not installed for the Python "
                "running this bridge. " + VOICE_INSTALL_HINT)
    return ("The " + pkg + " package is installed, but the Python running "
            "this bridge cannot load it: " + type(exc).__name__ + ": "
            + str(exc) + ". " + VOICE_INSTALL_HINT)


def _voice_decode_probe():
    # Build 6 (#221): availability by DECODE, not by import. The
    # import probe above can pass while transcription is dead:
    # faster-whisper 1.2.1's decode_audio passes a keyword
    # (metadata_errors) that PyAV 13+ removed from av.open(), so
    # with av 19 in the venv every clip died at decode — eight
    # days of "she can't hear me" that every import check called
    # healthy. This probe generates a quarter second of silence,
    # writes it as a WAV, and runs the REAL decode path
    # (faster_whisper.audio.decode_audio — the same call
    # transcription makes) over it. Cached like the import
    # probes: the verdict is per-interpreter, and an interpreter
    # does not change its packages mid-run. Returns
    # (ok, exception-or-None).
    key = "decode:faster_whisper"
    if key in _engine_probes:
        return _engine_probes[key]
    try:
        from faster_whisper.audio import decode_audio
        wav = _samples_to_wav([0.0] * 4000, 16000)
        fd, path = tempfile.mkstemp(suffix=".wav",
                                    prefix="navi-decode-probe-")
        try:
            with os.fdopen(fd, "wb") as f:
                f.write(wav)
            samples = decode_audio(path, sampling_rate=16000)
            n = len(samples)
        finally:
            try:
                os.unlink(path)
            except Exception:
                pass
        if n <= 0:
            raise RuntimeError(
                "the decoder returned no samples for a silent "
                "test clip")
        _engine_probes[key] = (True, None)
    except Exception as e:
        _engine_probes[key] = (False, e)
    return _engine_probes[key]


def _decode_detail(exc):
    # Why transcription is unavailable even though the package
    # imports — the sentence names the failing call and the
    # drift that causes it, with the cure (the pinned set).
    if exc is None:
        return None
    return ("faster-whisper is installed and imports, but it "
            "cannot decode audio in this Python: "
            + type(exc).__name__ + ": " + str(exc)
            + ". The speech packages have drifted apart — the "
            "known shape is a PyAV (the 'av' package) newer than "
            "faster-whisper's decoder expects; the voice "
            "requirements now pin av below 13 for exactly this. "
            "Cure: re-run the installer and say yes at the voice "
            "step — it repairs the venv to the pinned set — or "
            "run the venv's Python with: -m pip install "
            "-r requirements-voice.txt")


# -- CUDA execution probe (v0.4.1e, #229) ----------------------
# Device selection used to probe PRESENCE: torch reported a
# CUDA device on NeoArcadia, so Kokoro's pipeline loaded onto
# it — and the first real generation died with "CUDA error: no
# kernel image is available for execution on the device",
# because the installed wheels carry no kernels for the card's
# architecture (a Tesla V100, Volta, compute capability 7.0).
# The raw CUDA string was the operator-facing speech failure.
# This probe EXECUTES: a tiny real tensor operation on the
# device, synchronized, its result checked. The verdict is
# cached per interpreter, like the other voice probes, and a
# generation-time CUDA failure rewrites it (the self-healing
# path in synthesize_wav) so the next line goes to the CPU.


def _voice_cuda_probe():
    # Returns (state, info): state is "absent" (no torch, or
    # torch sees no CUDA device), "executing" (a real operation
    # ran on the device), or "failed" (a device is present but
    # execution died). info carries the device name, the compute
    # capability as "7.0"-style text, and the raw exception when
    # failed — the raw string is kept for the record, never used
    # as the operator's sentence (see _cuda_fallback_sentence).
    if "cuda-exec" in _engine_probes:
        return _engine_probes["cuda-exec"]
    verdict = ("absent", {})
    try:
        import torch
        if torch.cuda.is_available():
            info = {}
            try:
                info["name"] = str(torch.cuda.get_device_name(0))
            except Exception:
                info["name"] = "CUDA device"
            try:
                _maj, _min = torch.cuda.get_device_capability(0)
                info["capability"] = str(_maj) + "." + str(_min)
            except Exception:
                info["capability"] = ""
            try:
                _t = torch.zeros(8, device="cuda")
                _t = _t + 1
                torch.cuda.synchronize()
                if float(_t.sum().item()) != 8.0:
                    raise RuntimeError(
                        "the device returned a wrong result for a "
                        "trivial operation")
                verdict = ("executing", info)
            except Exception as e:
                info["error"] = type(e).__name__ + ": " + str(e)
                verdict = ("failed", info)
    except Exception:
        verdict = ("absent", {})
    _engine_probes["cuda-exec"] = verdict
    return verdict


def _cuda_fallback_sentence(info):
    # The operator's sentence for a fallen-back stage: the
    # device is named, its architecture is named, the
    # consequence is named — and the raw CUDA error never
    # appears in it. The probe's raw error rides the status as
    # a separate field, for the record.
    name = info.get("name") or "the graphics card"
    cap = info.get("capability") or ""
    return ("A graphics card is present (" + name
            + (", compute capability " + cap if cap else "")
            + "), but the installed speech wheels carry no "
              "kernels for its architecture, so real work on it "
              "fails. This stage runs on the CPU instead — fully "
              "working, a little slower to start.")


def _voice_stage_devices(vc):
    # Per-stage device verdicts, present / executing /
    # fallen-back, riding the same status surface as hearing
    # (#229: transcription and synthesis can disagree, so each
    # stage carries its own). Transcription's verdict is CPU by
    # construction: _stt_model loads faster-whisper on CPU int8
    # — no CUDA path exists on that stage to probe, and the
    # status says so rather than implying a choice was made.
    stt = {"device": "cpu", "state": "cpu",
           "detail": "Transcription runs on the CPU by design "
                     "(faster-whisper, int8); it never selects "
                     "the graphics card."}
    if vc["tts_engine"] != "kokoro":
        tts = {"device": "cpu", "state": "cpu",
               "detail": "Piper speaks on the CPU."}
        return {"stt": stt, "tts": tts}
    state, info = _voice_cuda_probe()
    if state == "executing":
        tts = {"device": "cuda", "state": "executing",
               "name": info.get("name") or "",
               "capability": info.get("capability") or "",
               "detail": None}
    elif state == "failed":
        tts = {"device": "cpu", "state": "fallen-back",
               "name": info.get("name") or "",
               "capability": info.get("capability") or "",
               "detail": _cuda_fallback_sentence(info),
               "error": info.get("error") or ""}
    else:
        tts = {"device": "cpu", "state": "cpu",
               "detail": "No CUDA device is present; speech runs "
                         "on the CPU."}
    return {"stt": stt, "tts": tts}


def _kokoro_device(vc):
    # The device Kokoro is ASKED for: CUDA only when the
    # execution probe proved the device can compute.
    state, _info = _voice_cuda_probe()
    return "cuda" if state == "executing" else "cpu"


def _is_cuda_error(exc):
    return "cuda" in (type(exc).__name__ + ": " + str(exc)).lower()


def _voice_cuda_mark_failed(exc):
    # A generation-time CUDA failure rewrites the probe verdict
    # and drops the cached pipeline: the next line is built on
    # the CPU. (The probe is a warmup-time truth; this keeps it
    # true after the field disagrees.)
    state, info = _voice_cuda_probe()
    info = dict(info)
    info["error"] = type(exc).__name__ + ": " + str(exc)
    _engine_probes["cuda-exec"] = ("failed", info)
    with _voice_lock:
        _voice_cache.pop("kokoro", None)


def _unavailable_message(label, pkg, exc):
    # What the loaders raise on failure — the same distinction the
    # availability probe reports, so a speak failure and the status
    # check tell one story.
    if not _module_available(pkg):
        return label + " is not installed. " + VOICE_INSTALL_HINT
    return (label + " is installed but failed to load: "
            + type(exc).__name__ + ": " + str(exc))


_KOKORO_VOICE_RE = re.compile(r"^([a-z])([fm])_([a-z0-9]+)$")
_KOKORO_LANGS = {
    "a": "American", "b": "British", "e": "Spanish", "f": "French",
    "h": "Hindi", "i": "Italian", "j": "Japanese",
    "p": "Brazilian Portuguese", "z": "Mandarin",
}
_voices_cache = {}


def _kokoro_voice_label(vid):
    # Labels are derived from the voice id (af_heart -> "Heart —
    # American female"), never a per-voice lookup table: a voice the
    # list has never seen still gets a correct name.
    m = _KOKORO_VOICE_RE.match(vid)
    if not m:
        return vid
    name = m.group(3).replace("_", " ").title()
    lang = _KOKORO_LANGS.get(m.group(1), "")
    who = "female" if m.group(2) == "f" else "male"
    return name + " — " + (lang + " " if lang else "") + who


def discover_kokoro_voices(current):
    # The voice picker is populated from what this machine's Kokoro
    # installation actually has — never a hardcoded list. Sources,
    # in order: the voice .pt files the installation already holds
    # (the kokoro package's own folder, voice/models, the
    # Hugging Face cache KPipeline downloads into); and, when none
    # are on disk yet, the voice files in the Kokoro-82M repo the
    # installed package itself pulls from (huggingface_hub ships
    # with kokoro) — those download on first use, the same pattern
    # the Piper voices follow below. When neither works (engine not
    # installed, machine offline), the saved voice alone is listed,
    # so the picker never renders empty or invents voices.
    if "kokoro" in _voices_cache:
        return _voices_cache["kokoro"]
    found = {}
    roots = []
    try:
        import kokoro as _kokoro_pkg
        roots.append(os.path.dirname(os.path.abspath(_kokoro_pkg.__file__)))
    except Exception:
        pass
    roots.append(VOICE_MODELS_DIR)
    hf_home = os.environ.get("HF_HOME") or os.path.join(
        os.path.expanduser("~"), ".cache", "huggingface")
    roots.append(os.path.join(hf_home, "hub"))
    for root in roots:
        if not root or not os.path.isdir(root):
            continue
        for dirpath, _dirs, files in os.walk(root):
            for fn in files:
                if fn.endswith(".pt"):
                    vid = fn[:-3]
                    if _KOKORO_VOICE_RE.match(vid):
                        found.setdefault(vid, "installed")
    source = "installed"
    if not found and _probe_engine("kokoro", "KPipeline")[0]:
        try:
            from huggingface_hub import list_repo_files
            for f in list_repo_files("hexgrad/Kokoro-82M"):
                if f.startswith("voices/") and f.endswith(".pt"):
                    vid = os.path.basename(f)[:-3]
                    if _KOKORO_VOICE_RE.match(vid):
                        found.setdefault(vid, "catalog")
            if found:
                source = "catalog"
        except Exception:
            pass
    if not found:
        found = {current: "current"}
        source = "current"
    result = {
        "voices": [
            {"id": vid, "label": _kokoro_voice_label(vid), "source": found[vid]}
            for vid in sorted(found)
        ],
        "source": source,
    }
    _voices_cache["kokoro"] = result
    return result


def discover_piper_voices(current):
    # Same rule for Piper: voices already downloaded into
    # voice/models/piper, plus the en voices in the piper-voices
    # repo (each downloads on first use — _piper_voice_file does
    # exactly that). The saved voice is always present.
    found = {}
    pdir = os.path.join(VOICE_MODELS_DIR, "piper")
    if os.path.isdir(pdir):
        for fn in sorted(os.listdir(pdir)):
            if fn.endswith(".onnx"):
                found[fn[:-5]] = "installed"
    if not any(s == "installed" for s in found.values()):
        try:
            from huggingface_hub import list_repo_files
            for f in list_repo_files("rhasspy/piper-voices"):
                parts = f.split("/")
                if (len(parts) == 5 and parts[0] == "en"
                        and parts[4].endswith(".onnx")):
                    found.setdefault(parts[4][:-5], "catalog")
        except Exception:
            pass
    if current and current not in found:
        found[current] = "current"
    source = ("installed" if any(s == "installed" for s in found.values())
              else "catalog" if any(s == "catalog" for s in found.values())
              else "current")
    voices = []
    for vid in sorted(found):
        parts = vid.split("-")
        if len(parts) == 3:
            label = (parts[1].title() + " (" + parts[0] + ", "
                     + parts[2] + ")")
        else:
            label = vid
        voices.append({"id": vid, "label": label, "source": found[vid]})
    return {"voices": voices, "source": source}


# Build 6 (#6, telemetry half): the last mic-state report a
# window made about ITSELF — the dashboard computes the reading
# for its Voice tab ("this window's own mic, this session") and,
# since this build, also posts it here on every push-to-talk
# attempt, so the field state finally exists in the bridge's log
# (start.log) instead of only on a screen someone must be
# watching. In-memory: it is a diagnostic, not a record.
_MIC_REPORT = {"state": "", "at": 0}


def voice_status(cfg):
    vc = voice_config(cfg)
    stt_ok, stt_exc = _probe_engine("faster_whisper", "WhisperModel")
    # Build 6 (#221): importable is not hearable — the decode
    # probe is the second half of the STT verdict. When the
    # import passes but the decode fails, hearing reads
    # unavailable and the detail names the decode, so the Voice
    # tab can never again call a deaf install healthy.
    dec_ok, dec_exc = (True, None)
    if stt_ok:
        dec_ok, dec_exc = _voice_decode_probe()
    engine_pkg = "kokoro" if vc["tts_engine"] == "kokoro" else "piper"
    tts_ok, tts_exc = _probe_engine(
        engine_pkg, "KPipeline" if engine_pkg == "kokoro" else "PiperVoice")
    disc = (discover_kokoro_voices(vc["tts_voice"])
            if vc["tts_engine"] == "kokoro"
            else discover_piper_voices(vc["tts_voice"]))
    # v0.4.1e (#229): per-stage device verdicts ride the same
    # surface — present / executing / fallen-back, with the
    # fallback's reason in the operator's words.
    _devs = _voice_stage_devices(vc)
    return {
        "enabled": vc["enabled"],
        "config": vc,
        "python": sys.executable,
        "venv": VOICE_VENV_SITE_PACKAGES,
        "stt": {
            "engine": "faster-whisper",
            "model": vc["stt_model"],
            "available": stt_ok and dec_ok,
            "detail": (_engine_detail("faster_whisper", stt_exc)
                       if not stt_ok else _decode_detail(dec_exc)),
            "decode_ok": dec_ok,
            "device": _devs["stt"],
        },
        "mic_report": dict(_MIC_REPORT),
        "tts": {
            "engine": vc["tts_engine"],
            "voice": vc["tts_voice"],
            "available": tts_ok,
            "detail": _engine_detail(engine_pkg, tts_exc),
            "device": _devs["tts"],
        },
        "voices": disc["voices"],
        "voices_source": disc["source"],
        "call": load_voice_state().get("call"),
    }


def load_voice_state():
    try:
        with open(VOICE_STATE_PATH) as f:
            data = json.load(f)
        if isinstance(data, dict):
            return data
    except Exception:
        pass
    return {"call": None}


def save_voice_state(state):
    os.makedirs(VOICE_DIR, exist_ok=True)
    with open(VOICE_STATE_PATH, "w") as f:
        json.dump(state, f)


def load_voice_calls():
    try:
        with open(VOICE_CALLS_PATH) as f:
            data = json.load(f)
        if isinstance(data, list):
            return data
    except Exception:
        pass
    return []


_voice_cache = {}
_voice_lock = threading.Lock()


def _stt_model(vc):
    # Loaded once, then kept warm: a Whisper load takes seconds and
    # a call shouldn't pay it per sentence. The download lands in
    # voice/models/ on first use and stays there across installs.
    key = "stt:" + vc["stt_model"]
    with _voice_lock:
        if key not in _voice_cache:
            try:
                from faster_whisper import WhisperModel
            except Exception as e:
                raise VoiceUnavailable(_unavailable_message("faster-whisper", "faster_whisper", e))
            os.makedirs(VOICE_MODELS_DIR, exist_ok=True)
            _voice_cache[key] = WhisperModel(
                vc["stt_model"], device="cpu", compute_type="int8",
                download_root=os.path.join(VOICE_MODELS_DIR, "whisper"),
            )
        return _voice_cache[key]


def transcribe_audio(audio_bytes, vc):
    if not audio_bytes:
        raise VoiceUnavailable("No audio arrived — the clip was empty.")
    # Build 6 (#221): the decode verdict gates the attempt — a
    # broken decoder is named here, in the operator's words,
    # instead of surfacing as a raw TypeError from inside the
    # model's transcribe call (which read as a dead microphone).
    dec_ok, dec_exc = _voice_decode_probe()
    if not dec_ok:
        raise VoiceUnavailable(_decode_detail(dec_exc))
    model = _stt_model(vc)
    fd, path = tempfile.mkstemp(suffix=".webm", prefix="navi-stt-")
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(audio_bytes)
        segments, _info = model.transcribe(path, beam_size=1, vad_filter=True)
        return "".join(seg.text for seg in segments).strip()
    finally:
        try:
            os.unlink(path)
        except Exception:
            pass


def _samples_to_wav(samples, sample_rate):
    # Float samples in [-1, 1] to a 16-bit PCM WAV, stdlib only.
    import wave as wave_mod
    buf = io.BytesIO()
    with wave_mod.open(buf, "wb") as w:
        w.setnchannels(1)
        w.setsampwidth(2)
        w.setframerate(int(sample_rate))
        frames = bytearray()
        for s in samples:
            v = max(-1.0, min(1.0, float(s)))
            frames += struct.pack("<h", int(v * 32767))
        w.writeframes(bytes(frames))
    return buf.getvalue()


def _pitch_shift_wav(wav_bytes, vc):
    # Varispeed pitch: replaying the same frames at a scaled rate
    # shifts pitch (and tempo along with it) — the honest stdlib
    # way of doing it, no DSP library pretending otherwise.
    try:
        factor = float(vc.get("pitch") or 1.0)
    except (TypeError, ValueError):
        factor = 1.0
    if abs(factor - 1.0) < 0.001:
        return wav_bytes
    import wave as wave_mod
    src = wave_mod.open(io.BytesIO(wav_bytes), "rb")
    params = src.getparams()
    frames = src.readframes(src.getnframes())
    src.close()
    out = io.BytesIO()
    dst = wave_mod.open(out, "wb")
    dst.setparams(params)
    dst.setframerate(max(4000, int(params.framerate * factor)))
    dst.writeframes(frames)
    dst.close()
    return out.getvalue()


def _kokoro_pipeline(vc):
    key = "kokoro"
    with _voice_lock:
        if key not in _voice_cache:
            try:
                from kokoro import KPipeline
            except Exception as e:
                raise VoiceUnavailable(_unavailable_message("Kokoro", "kokoro", e))
            lang = "b" if vc["tts_voice"].startswith("b") else "a"
            # v0.4.1e (#229): the pipeline is built on the
            # device the EXECUTION probe proved — never on mere
            # presence. A Kokoro too old to take a device
            # argument keeps its default, and when that default
            # would be a proven-broken card, the loaded model is
            # moved to the CPU after construction.
            device = _kokoro_device(vc)
            try:
                _voice_cache[key] = KPipeline(lang_code=lang,
                                              device=device)
            except TypeError:
                _voice_cache[key] = KPipeline(lang_code=lang)
                if device == "cpu":
                    try:
                        _model = getattr(_voice_cache[key],
                                         "model", None)
                        if _model is not None:
                            _model.to("cpu")
                    except Exception:
                        pass
        return _voice_cache[key]


PIPER_VOICE_BASE = "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/"


def _piper_voice_file(vc):
    # Piper voices are .onnx + .onnx.json pairs named like
    # en_US-lessac-medium, filed as en/<locale>/<name>/<quality>/.
    name = vc["tts_voice"]
    parts = name.split("-")
    if len(parts) != 3:
        raise VoiceUnavailable("Unknown Piper voice name: " + name)
    locale, speaker, quality = parts
    lang = locale.split("_")[0]
    rel = lang + "/" + locale + "/" + speaker + "/" + quality + "/" + name
    onnx = os.path.join(VOICE_MODELS_DIR, "piper", name + ".onnx")
    conf = onnx + ".json"
    if not (os.path.isfile(onnx) and os.path.isfile(conf)):
        os.makedirs(os.path.dirname(onnx), exist_ok=True)
        for suffix, dest in ((".onnx", onnx), (".onnx.json", conf)):
            try:
                urllib.request.urlretrieve(PIPER_VOICE_BASE + rel + suffix, dest)
            except Exception as e:
                raise VoiceUnavailable("Could not download the Piper voice " + name + ": " + str(e))
    return onnx


def _piper_load(vc):
    # Load (or fetch from cache) the configured Piper voice — the
    # warmup path and synthesis share this one cache entry.
    try:
        from piper import PiperVoice
    except Exception as e:
        raise VoiceUnavailable(_unavailable_message("Piper", "piper", e))
    onnx = _piper_voice_file(vc)
    key = "piper:" + onnx
    with _voice_lock:
        if key not in _voice_cache:
            _voice_cache[key] = PiperVoice.load(onnx)
        return _voice_cache[key]


def _piper_wav(text, vc):
    voice = _piper_load(vc)
    import wave as wave_mod
    buf = io.BytesIO()
    try:
        from piper.config import SynthesisConfig
        with wave_mod.open(buf, "wb") as w:
            voice.synthesize_wav(
                text, w,
                syn_config=SynthesisConfig(length_scale=1.0 / max(0.25, float(vc["rate"]))),
            )
    except VoiceUnavailable:
        raise
    except Exception as e:
        raise VoiceUnavailable("Piper could not speak that line: " + str(e))
    return _pitch_shift_wav(buf.getvalue(), vc)


def synthesize_wav(text, vc):
    text = (text or "").strip()
    if not text:
        raise VoiceUnavailable("Nothing to say — the line was empty.")
    if not vc["enabled"]:
        raise VoiceUnavailable("Voice is turned off — enable it in the Voice tab.")
    if vc["tts_engine"] == "piper":
        return _piper_wav(text, vc)
    pipeline = _kokoro_pipeline(vc)
    samples = []
    try:
        for _g, _p, audio in pipeline(text, voice=vc["tts_voice"], speed=float(vc["rate"])):
            if audio is None:
                continue
            try:
                samples.extend(float(x) for x in audio)
            except TypeError:
                samples.append(float(audio))
    except VoiceUnavailable:
        raise
    except Exception as e:
        # v0.4.1e (#229): a CUDA failure at generation time —
        # the probe said the card could compute and the field
        # disagreed — is named in the operator's words, never
        # relayed raw, and it rewrites the probe verdict so the
        # next line is built on the CPU.
        if _is_cuda_error(e):
            _state, _info = _voice_cuda_probe()
            _voice_cuda_mark_failed(e)
            raise VoiceUnavailable(
                "Her voice could not use the graphics card for "
                "that line. " + _cuda_fallback_sentence(_info)
                + " The next line will be spoken from the CPU.")
        raise VoiceUnavailable("Kokoro could not speak that line: "
                               + str(e))
    if not samples:
        raise VoiceUnavailable("Kokoro produced no audio for that line.")
    return _pitch_shift_wav(_samples_to_wav(samples, 24000), vc)


def speak_reply(text, cfg):
    # /api/chat with speak:true — her reply out loud on this machine
    # as well as in the thread. Best-effort by design: the reply is
    # already on its way back to the dashboard either way, so a
    # speech failure reports itself instead of failing the chat.
    vc = voice_config(cfg)
    if not vc["enabled"]:
        return {"ok": False, "error": "Voice is turned off."}
    try:
        wav = synthesize_wav(text, vc)
    except VoiceUnavailable as e:
        return {"ok": False, "error": str(e)}
    except Exception as e:
        return {"ok": False, "error": "Speech failed: " + str(e)}
    try:
        import sounddevice as sd
        import soundfile as sf
        data, rate = sf.read(io.BytesIO(wav), dtype="float32")
        sd.play(data, rate)
        return {"ok": True, "via": "sounddevice"}
    except Exception:
        pass
    fd, path = tempfile.mkstemp(suffix=".wav", prefix="navi-say-")
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(wav)
        for player in ("aplay", "paplay", "afplay"):
            exe = shutil.which(player)
            if exe:
                subprocess.Popen([exe, path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                return {"ok": True, "via": player}
        return {"ok": False, "error": "No audio player found (sounddevice, aplay, paplay, or afplay)."}
    except Exception as e:
        return {"ok": False, "error": "Playback failed: " + str(e)}


def llamafile_alive(endpoint):
    try:
        urllib.request.urlopen(endpoint.replace("/v1", "/"), timeout=2)
        return True
    except Exception:
        return False


_llamafile_proc = None


def llamafile_port(endpoint):
    try:
        port = urllib.parse.urlparse(endpoint).port
    except Exception:
        port = None
    return port or 8080


def llamafile_binary(cfg):
    # Where the installer puts navi.llamafile: the configured install
    # dir, or right beside this bridge — the bridge ships in the
    # install root, so on a normal install those are the same place.
    dirs = []
    install_dir = cfg.get("install_dir")
    if install_dir:
        dirs.append(os.path.expanduser(install_dir))
    dirs.append(ROOT)
    for d in dirs:
        for name in ("navi.llamafile", "navi.llamafile.exe"):
            path = os.path.join(d, name)
            if os.path.isfile(path):
                return path
    return None


def wait_for_llamafile(endpoint, proc, log_path=None):
    # Loading a multi-GB model takes a while; give it two minutes.
    for _ in range(120):
        if llamafile_alive(endpoint):
            return True, None
        if proc is not None and proc.poll() is not None:
            tail = ""
            if log_path:
                try:
                    with open(log_path, "rb") as f:
                        data = f.read().decode(errors="replace").strip()
                    if data:
                        tail = data.splitlines()[-1]
                except Exception:
                    tail = ""
            msg = ("navi.llamafile exited (code " + str(proc.returncode) +
                   ") before it started answering.")
            if tail:
                msg += " Last log line: " + tail
            msg += " Full log: navi.llamafile.log beside the app."
            return False, msg
        time.sleep(1)
    return False, ("navi.llamafile is still loading after two minutes — "
                   "its output is in navi.llamafile.log beside the app.")


def hardware_tier_name(detected):
    # The plain tier names (Build 14, #157): "potato" / "standard"
    # / "beast", derived from the detected hardware alone. These
    # sit alongside the derivation table's own tier words (fluid /
    # balanced / performance / workstation / constrained, kept in
    # the stored profile and its tierLabel) — this class is the
    # one the warm-up default and the profile view speak in.
    # Beast is judged first: a small-core-count machine with a
    # real GPU or workstation RAM is not a potato.
    d = detected if isinstance(detected, dict) else {}
    try:
        ram = float(d.get("totalRamGB") or 0)
    except Exception:
        ram = 0.0
    try:
        cores = int(d.get("cpuCores") or 0)
    except Exception:
        cores = 0
    try:
        vram = float(d.get("gpuVramGB") or 0)
    except Exception:
        vram = 0.0
    discrete = d.get("gpuKind") == "discrete"
    if (discrete and vram >= 8) or ram >= 32:
        return "beast"
    if (ram <= 12 and not discrete) or (cores and cores <= 4):
        return "potato"
    return "standard"


def _warm_at_value(raw):
    # runtime.warm_at, validated: a "HH:MM" 24h string, else None
    # (absent / null / malformed all read as off) (Build 14, #157).
    if not isinstance(raw, str):
        return None
    raw = raw.strip()
    parts = raw.split(":")
    if len(parts) != 2:
        return None
    try:
        hh = int(parts[0]); mm = int(parts[1])
    except Exception:
        return None
    if not (0 <= hh <= 23 and 0 <= mm <= 59):
        return None
    return "%02d:%02d" % (hh, mm)


def runtime_settings(cfg):
    # Her llamafile runtime settings, from the hardware profile the
    # installer wrote into the config (hardware_profile + runtime).
    # No profile (a pre-.34 config) -> the fluid floor.
    rt = cfg.get("runtime") or {}
    hp = (cfg.get("hardware_profile") or {}).get("settings") or {}
    def pick(a, b, fallback):
        v = rt.get(a)
        if v is None:
            v = hp.get(b)
        try:
            return int(v)
        except Exception:
            return fallback
    # Warm-up (Build 14, #157): runtime.warmup always wins when
    # the operator set it; unset, warm-up is on for the potato and
    # standard tiers and off for beast (a beast loads fast enough
    # that the dead air is not the problem). No profile at all —
    # the fluid floor, a modest machine by definition — reads as
    # on. runtime.warm_at is the scheduled warm ("HH:MM" or off).
    if rt.get("warmup") is not None:
        warmup = bool(rt.get("warmup"))
    else:
        detected = (cfg.get("hardware_profile") or {}).get("detected")
        warmup = (not isinstance(detected, dict)
                  or hardware_tier_name(detected) in ("potato", "standard"))
    return {
        "threads": pick("threads", "threads", 4),
        "ctx_size": pick("ctx_size", "ctxSize", 4096),
        "batch_size": pick("batch_size", "batchSize", 512),
        "gpu_layers": pick("gpu_layers", "gpuLayers", 0),
        "warmup": warmup,
        "warm_at": _warm_at_value(rt.get("warm_at")),
    }


def llamafile_args(cfg):
    s = runtime_settings(cfg)
    args = ["--server", "--v2", "--ctx-size", str(s["ctx_size"]),
            "--threads", str(s["threads"]), "--batch-size", str(s["batch_size"])]
    if s["gpu_layers"] > 0:
        args += ["--n-gpu-layers", str(s["gpu_layers"])]
    return args


def detect_hardware():
    # Real detection for /api/hardware/redetect — stdlib only, per-OS.
    # Linux: /proc + lspci/nvidia-smi. macOS: sysctl + system_profiler.
    # Windows: CIM via PowerShell. Anything unconfirmable stays
    # "unknown" rather than guessed — derivation treats it CPU-only.
    import platform as _pf
    det = {"os": "unknown", "osLabel": _pf.platform(), "cpuModel": _pf.processor() or "unknown CPU",
           "cpuCores": os.cpu_count() or 1, "performanceCores": None,
           "totalRamGB": 0.0, "gpuPresent": False, "gpuName": "", "gpuKind": "none", "gpuVramGB": None}
    system = _pf.system().lower()
    try:
        if system == "linux":
            det["os"] = "linux"; det["osLabel"] = "Linux"
            mem_kb = 0
            cpu_name = ""
            with open("/proc/meminfo") as f:
                for line in f:
                    if line.startswith("MemTotal:"):
                        mem_kb = int(line.split()[1])
            with open("/proc/cpuinfo") as f:
                for line in f:
                    if line.startswith("model name"):
                        cpu_name = line.split(":", 1)[1].strip(); break
            if cpu_name:
                det["cpuModel"] = cpu_name
            det["totalRamGB"] = round(mem_kb / 1048576.0, 1) if mem_kb else 0.0
            # Intel hybrid: performance cores are hard to prove from
            # /proc alone; the derivation's logical-core rule covers it.
            try:
                out = subprocess.run(["nvidia-smi", "--query-gpu=name,memory.total", "--format=csv,noheader"],
                                     capture_output=True, text=True, timeout=5).stdout.strip()
                if out:
                    parts = [p.strip() for p in out.splitlines()[0].split(",")]
                    det["gpuPresent"] = True; det["gpuName"] = parts[0]; det["gpuKind"] = "discrete"
                    if len(parts) > 1:
                        det["gpuVramGB"] = int(round(float(parts[1].split()[0]) / 1024.0))
                else:
                    raise RuntimeError("no nvidia-smi output")
            except Exception:
                try:
                    out = subprocess.run(["lspci"], capture_output=True, text=True, timeout=5).stdout
                    for line in out.splitlines():
                        if "VGA compatible controller" in line or "3D controller" in line:
                            det["gpuPresent"] = True
                            det["gpuName"] = line.split(":", 2)[-1].strip()
                            low = det["gpuName"].lower()
                            det["gpuKind"] = "discrete" if ("nvidia" in low or "radeon rx" in low or "geforce" in low) else "integrated"
                            break
                except Exception:
                    pass
        elif system == "darwin":
            det["os"] = "macos"; det["osLabel"] = "macOS"
            def _sysctl(key):
                return subprocess.run(["sysctl", "-n", key], capture_output=True, text=True, timeout=5).stdout.strip()
            det["cpuModel"] = _sysctl("machdep.cpu.brand_string") or det["cpuModel"]
            det["cpuCores"] = int(_sysctl("hw.ncpu") or det["cpuCores"])
            try:
                det["performanceCores"] = int(_sysctl("hw.perflevel0.logicalcpu") or 0) or None
            except Exception:
                det["performanceCores"] = None
            det["totalRamGB"] = round(int(_sysctl("hw.memsize") or 0) / 1073741824.0, 1)
            det["gpuPresent"] = True; det["gpuName"] = det["cpuModel"] + " GPU"; det["gpuKind"] = "integrated"
        elif system == "windows":
            det["os"] = "windows"; det["osLabel"] = "Windows"
            def _ps(expr):
                return subprocess.run(["powershell", "-NoProfile", "-Command", expr],
                                      capture_output=True, text=True, timeout=10).stdout.strip()
            det["cpuModel"] = _ps("(Get-CimInstance Win32_Processor | Select-Object -First 1).Name") or det["cpuModel"]
            det["cpuCores"] = int(_ps("(Get-CimInstance Win32_Processor | Select-Object -First 1).NumberOfLogicalProcessors") or det["cpuCores"])
            det["totalRamGB"] = round(int(_ps("(Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory") or 0) / 1073741824.0, 1)
            gpu_name = _ps("(Get-CimInstance Win32_VideoController | Select-Object -First 1).Name")
            if gpu_name:
                det["gpuPresent"] = True; det["gpuName"] = gpu_name
                low = gpu_name.lower()
                if "nvidia" in low or "geforce" in low or "radeon rx" in low:
                    det["gpuKind"] = "discrete"
                    vram = _ps("(Get-CimInstance Win32_VideoController | Select-Object -First 1).AdapterRAM")
                    try:
                        det["gpuVramGB"] = int(round(int(vram) / 1073741824.0)) or None
                    except Exception:
                        det["gpuVramGB"] = None
                else:
                    det["gpuKind"] = "integrated"
    except Exception:
        pass
    return det


def derive_hardware_settings(d):
    # The bridge's copy of the derivation table (hardware.ts is the
    # source of truth; the installers carry the same table in shell).
    ram = float(d.get("totalRamGB") or 0)
    logical = max(1, int(d.get("cpuCores") or 1))
    perf = d.get("performanceCores")
    reasoning = []
    if perf:
        threads = max(2, min(12, int(perf)))
        reasoning.append(f"Threads {threads}: matched to the {perf} performance cores.")
    elif logical >= 16:
        threads = 8; reasoning.append(f"Threads 8: half of {logical} logical cores, capped.")
    elif logical >= 8:
        threads = 4; reasoning.append(f"Threads 4: half of {logical} logical cores.")
    elif logical >= 4:
        threads = 2; reasoning.append(f"Threads 2: conservative for a {logical}-core machine.")
    else:
        threads = 1; reasoning.append("Threads 1: single-core floor.")
    if ram < 6:
        ctx = PROMPT_FLOOR_CTX; reasoning.append(f"Context {PROMPT_FLOOR_CTX}: under 6 GiB RAM — floored at the product's prompt floor (#310): a smaller window cannot hold her standing prompt.")
    elif ram < 12:
        ctx = 4096; reasoning.append(f"Context 4096: {ram:.1f} GiB RAM — the proven fluid size for this class.")
    elif ram < 20:
        ctx = 8192; reasoning.append(f"Context 8192: {ram:.1f} GiB RAM leaves comfortable headroom.")
    elif ram < 40:
        ctx = 16384; reasoning.append(f"Context 16384: {ram:.1f} GiB RAM.")
    else:
        ctx = 32768; reasoning.append(f"Context 32768: {ram:.1f} GiB RAM — workstation headroom.")
    batch = 256 if ram < 6 else (512 if ram < 20 else 1024)
    reasoning.append(f"Batch {batch}: sized to RAM ({ram:.1f} GiB).")
    gpu_layers = 0
    vram = d.get("gpuVramGB")
    if d.get("gpuKind") == "discrete" and vram is not None and vram >= 4:
        gpu_layers = 99 if vram >= 12 else (28 if vram >= 8 else (20 if vram >= 6 else 12))
        reasoning.append(f"GPU layers {gpu_layers}: discrete GPU with {vram} GiB VRAM.")
    elif d.get("gpuKind") == "integrated":
        reasoning.append("GPU layers 0: integrated graphics share system RAM — CPU inference is the fluid path.")
    else:
        reasoning.append("GPU layers 0: no confirmed discrete GPU — CPU-only.")
    if ram < 6 or logical <= 4:
        tier, tier_label = "constrained", "Constrained — small machine, fluid floor"
    elif gpu_layers > 0 and ram >= 16:
        tier, tier_label = ("workstation", "Workstation — discrete GPU + ample RAM") if ram >= 32 else ("performance", "Performance — discrete GPU acceleration")
    elif ram >= 20:
        tier, tier_label = "performance", "Performance — ample RAM, CPU inference scaled up"
    elif ram >= 12:
        tier, tier_label = "balanced", "Balanced — above the fluid baseline"
    else:
        tier, tier_label = "fluid", "Fluid — Framework-class baseline"
    return {"threads": threads, "ctxSize": ctx, "batchSize": batch, "gpuLayers": gpu_layers}, tier, tier_label, reasoning


def _profile_source(hp):
    # Who wrote this hardware profile (Build 14, #157), as best
    # the config itself can say: the startup auto-profile pass
    # marks its block with auto=True and only that block reads as
    # "auto"; a Settings override is the operator's; anything else
    # with a profile is the installer's — pre-Build-14 profiles
    # wrote source "auto" for both the installer and a Settings
    # re-detect, and those two cannot be told apart after the
    # fact, so they read as "installer", their original writer.
    if hp.get("auto") is True:
        return "auto"
    src = str(hp.get("source") or "")
    if src == "manual":
        return "operator"
    if src in ("auto", "installer"):
        return "installer"
    return None


def hardware_profile_view(cfg):
    hp = cfg.get("hardware_profile")
    if isinstance(hp, dict) and hp.get("detected") and hp.get("settings"):
        view = dict(hp)
        s = runtime_settings(cfg)
        view["settings"] = {"threads": s["threads"], "ctxSize": s["ctx_size"], "batchSize": s["batch_size"], "gpuLayers": s["gpu_layers"]}
        view["flags"] = " ".join(llamafile_args(cfg)[2:])
        # Tier naming (Build 14, #157): the view's "tier" is the
        # plain potato / standard / beast class derived from the
        # detected hardware; the derivation table's own tier word
        # stays readable alongside it as "derivedTier" (and in
        # tierLabel, which the dashboard already shows).
        view["derivedTier"] = hp.get("tier")
        view["tier"] = hardware_tier_name(hp.get("detected") or {})
        view["profileSource"] = _profile_source(hp)
        return view
    return {"detected": None, "settings": runtime_settings(cfg), "source": "auto",
            "tier": None, "profileSource": None,
            "note": "No hardware profile in this config (it predates build .34) — the fluid floor is in use. Re-detect to profile this machine."}


def auto_profile_on_startup(cfg):
    # Automatic profiling (Build 14, #157): a config with NO
    # hardware profile used to run the fluid floor (4096/4/512)
    # forever — the machine never profiled itself. On bridge
    # startup, profile it once: detect + derive, persist the
    # profile, and fill derived runtime defaults ONLY where the
    # runtime keys are absent — operator-set runtime values are
    # never overridden. Any failure leaves the floor behavior
    # exactly as it was before this pass existed.
    try:
        hp = cfg.get("hardware_profile") if isinstance(cfg, dict) else None
        if isinstance(hp, dict) and hp.get("detected") and hp.get("settings"):
            return cfg
        detected = detect_hardware()
        settings, tier, tier_label, reasoning = derive_hardware_settings(detected)
        tier_name = hardware_tier_name(detected)
        now = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        profile = {"detected": detected, "settings": settings, "autoSettings": settings,
                   "source": "auto", "auto": True, "tier": tier, "tierLabel": tier_label,
                   "reasoning": reasoning, "detectedAt": now,
                   "installLog": [f"[hardware] {now} — automatic profiling on bridge startup (Build 14, #157): this config carried no hardware profile",
                                  f"[hardware] CPU: {detected['cpuModel']} — {detected['cpuCores']} logical cores",
                                  f"[hardware] RAM: {detected['totalRamGB']} GiB",
                                  f"[hardware] GPU: {detected['gpuName'] or 'none'} ({detected['gpuKind']})",
                                  f"[hardware] Tier class: {tier_name}"] + [f"[hardware] → {r}" for r in reasoning]}
        fresh = load_config()
        hp2 = fresh.get("hardware_profile") if isinstance(fresh, dict) else None
        if isinstance(hp2, dict) and hp2.get("detected") and hp2.get("settings"):
            return fresh
        fresh["hardware_profile"] = profile
        rt = fresh.get("runtime") if isinstance(fresh.get("runtime"), dict) else {}
        rt = dict(rt)
        defaults = {"threads": settings["threads"], "ctx_size": settings["ctxSize"],
                    "batch_size": settings["batchSize"], "gpu_layers": settings["gpuLayers"]}
        filled = []
        for key, val in defaults.items():
            if rt.get(key) is None:
                rt[key] = val
                filled.append(key)
        if rt.get("flags") is None and len(filled) == len(defaults):
            rt["flags"] = (f"--ctx-size {settings['ctxSize']} --threads {settings['threads']} "
                           f"--batch-size {settings['batchSize']}"
                           + (f" --n-gpu-layers {settings['gpuLayers']}" if settings["gpuLayers"] > 0 else ""))
        fresh["runtime"] = rt
        if not save_config(fresh):
            print("Hardware profile: this config had none saved and the automatic "
                  "profile could not be written to the config file — the fluid "
                  "floor settings stay in use for this run.", flush=True)
            return cfg
        try:
            with open(os.path.join(ROOT, "hardware-profile.log"), "a") as f:
                for line in profile.get("installLog") or []:
                    f.write(line + "\n")
        except Exception:
            pass
        gpu_words = (("discrete GPU " + str(detected.get("gpuName") or "")) if detected.get("gpuKind") == "discrete"
                     else ("integrated graphics" if detected.get("gpuKind") == "integrated" else "no discrete GPU"))
        print("Hardware profile: this config had none saved, so Apex profiled this "
              "machine on startup and saved the profile — " + tier_name + " tier: "
              + str(detected.get("totalRamGB")) + " GiB RAM, "
              + str(detected.get("cpuCores")) + " CPU cores, " + gpu_words + ". "
              "Navi will run with context " + str(settings["ctxSize"])
              + ", threads " + str(settings["threads"])
              + ", batch " + str(settings["batchSize"])
              + ". Any runtime settings you set yourself were left untouched.",
              flush=True)
        return fresh
    except Exception as e:
        print("Hardware profile: automatic profiling on startup did not run ("
              + str(e) + ") — the fluid floor settings stay in use, exactly as before.",
              flush=True)
        return cfg


def save_hardware_profile(profile):
    cfg = load_config()
    # Warm-up keys round-trip (Build 14, #157): a re-detect or an
    # override rewrites the runtime block from the profile below,
    # but runtime.warmup / runtime.warm_at are operator choices
    # that live only in that block — capture them before the
    # rewrite and carry them over, never drop them.
    _prev_rt = cfg.get("runtime") if isinstance(cfg.get("runtime"), dict) else {}
    cfg["hardware_profile"] = profile
    s = profile["settings"]
    cfg["runtime"] = {"threads": s["threads"], "ctx_size": s["ctxSize"], "batch_size": s["batchSize"], "gpu_layers": s["gpuLayers"],
                      "flags": f"--ctx-size {s['ctxSize']} --threads {s['threads']} --batch-size {s['batchSize']}" + (f" --n-gpu-layers {s['gpuLayers']}" if s["gpuLayers"] > 0 else "")}
    for _k in ("warmup", "warm_at"):
        if _k in _prev_rt:
            cfg["runtime"][_k] = _prev_rt[_k]
    with open(CONFIG_PATH, "w") as f:
        json.dump(cfg, f, indent=2)
    try:
        with open(os.path.join(ROOT, "hardware-profile.log"), "a") as f:
            for line in profile.get("installLog") or []:
                f.write(line + "\n")
    except Exception:
        pass
    return cfg


def _warmup_completion(endpoint, cfg):
    # One throwaway completion (Build 14, #157): the same chat
    # completions shape llamafile_chat builds, capped at a single
    # token, with a one-word prompt that is never shown anywhere.
    # Its only job is to make the model do its first real work —
    # page in, caches hot — before the operator's first ask, so
    # the first-ask dead air happens here instead. A failure is a
    # log line, never an error: warm-up is a courtesy, not a gate.
    try:
        name = served_model(endpoint) or (cfg.get("live") or {}).get("model_name") or "navi"
        body = json.dumps({
            "model": name,
            "messages": [{"role": "user", "content": "Hi"}],
            "stream": False,
            "max_tokens": 1,
        }).encode()
        req = urllib.request.Request(
            endpoint.rstrip("/") + "/chat/completions",
            data=body,
            headers={"Content-Type": "application/json"},
        )
        with urllib.request.urlopen(req, timeout=120) as resp:
            resp.read()
        print("Warm-up: navi.llamafile answered a one-token warm-up "
              "completion — the model is warm for the first ask.", flush=True)
        return True
    except Exception as e:
        print("Warm-up: the warm-up completion did not finish (" + str(e)
              + ") — Navi will simply warm up on the first ask instead.",
              flush=True)
        return False


def _maybe_warmup_llamafile(cfg, endpoint):
    # Fire the warm-up on its own daemon thread (Build 14, #157):
    # the start that triggered it has already reported healthy and
    # must not wait on a generation, however small. Whether it
    # fires at all is runtime_settings' call (explicit
    # runtime.warmup wins; the tier decides when it is unset).
    try:
        if not runtime_settings(cfg)["warmup"]:
            return False
    except Exception:
        return False
    threading.Thread(target=_warmup_completion, args=(endpoint, cfg),
                     daemon=True).start()
    return True


def _warm_at_loop():
    # Scheduled warm (Build 14, #157): a lightweight timer that
    # checks once a minute for runtime.warm_at ("HH:MM", 24h;
    # absent/null = off). At the configured minute, a llamafile
    # that is not answering is started through the same start
    # path the dashboard's Connect uses — and that fresh start
    # warms it via the warm-on-start hook above. One fire per
    # configured minute per day. (Looked for an existing minute-
    # cadence thread to ride first: the backup loop sleeps 30
    # minutes and the update loop 3 hours, so this is its own.)
    fired_for = ""
    while True:
        time.sleep(60)
        try:
            cfg = load_config()
            warm_at = runtime_settings(cfg)["warm_at"]
            if not warm_at:
                continue
            now = time.localtime()
            if time.strftime("%H:%M", now) != warm_at:
                continue
            stamp = time.strftime("%Y-%m-%d", now) + " " + warm_at
            if fired_for == stamp:
                continue
            fired_for = stamp
            endpoint = cfg.get("live", {}).get("llamafile_endpoint", "http://localhost:8080/v1")
            if llamafile_alive(endpoint):
                continue
            if cfg.get("startup", "on-ui-launch") == "on-boot":
                # Boot mode owns the server (same rule as Connect):
                # the bridge never spawns a second copy behind the
                # service's back, scheduled or not.
                print("Scheduled warm (" + warm_at + "): navi.llamafile is not "
                      "answering, but this install starts it at system boot — "
                      "leaving it to the boot service.", flush=True)
                continue
            print("Scheduled warm (" + warm_at + "): navi.llamafile is not "
                  "answering — starting and warming it now.", flush=True)
            ok, error = start_llamafile(cfg, endpoint)
            if not ok:
                print("Scheduled warm (" + warm_at + "): the start did not "
                      "come up: " + str(error), flush=True)
        except Exception:
            pass


def start_llamafile(cfg, endpoint):
    # App-load mode owns the server process: the bridge starts it
    # with the app's one launch command — ./navi.llamafile --server
    # --v2 plus the runtime flags from the hardware profile in the
    # config (see runtime_settings) — never a hard-coded set.
    # Returns (ok, error). Output goes to navi.llamafile.log next to
    # the binary so a failed start is diagnosable, not silent.
    global _llamafile_proc
    if _llamafile_proc is not None and _llamafile_proc.poll() is None:
        return wait_for_llamafile(endpoint, _llamafile_proc)
    binary = llamafile_binary(cfg)
    if not binary:
        searched = os.path.expanduser(cfg.get("install_dir") or ROOT)
        return False, ("navi.llamafile was not found in " + searched +
                       " — run the installer so it can download the model server.")
    log_path = os.path.join(os.path.dirname(binary), "navi.llamafile.log")
    try:
        log = open(log_path, "ab")
    except Exception:
        log = subprocess.DEVNULL
    try:
        _llamafile_proc = subprocess.Popen(
            [binary] + llamafile_args(cfg),
            cwd=os.path.dirname(binary),
            stdout=log,
            stderr=subprocess.STDOUT,
            start_new_session=True,
        )
    except Exception as e:
        return False, "Could not start navi.llamafile: " + str(e)
    ok, error = wait_for_llamafile(endpoint, _llamafile_proc, log_path)
    if ok:
        # Warm-on-start (Build 14, #157): this bridge just started
        # the server and it reports healthy — fire ONE warm-up
        # completion on a daemon thread so the model-load dead air
        # is spent now, not on the operator's first ask. Only the
        # fresh-start path warms: the already-running return at the
        # top of this function is a probe, not a start.
        _maybe_warmup_llamafile(cfg, endpoint)
    return ok, error


_served_model_cache = {}


def served_model(endpoint):
    # The model id the server itself reports (GET /v1/models). Asking
    # beats assuming: the name baked into a llamafile is whatever its
    # builder chose, and a server that validates the model string
    # rejects a guessed one. Cached per endpoint; None when the
    # server won't say (callers fall back to the configured name).
    if endpoint in _served_model_cache:
        return _served_model_cache[endpoint]
    found = None
    try:
        with urllib.request.urlopen(endpoint.rstrip("/") + "/models", timeout=5) as resp:
            data = json.loads(resp.read().decode())
        entries = data.get("data") or []
        if entries and entries[0].get("id"):
            found = str(entries[0]["id"])
    except Exception:
        found = None
    _served_model_cache[endpoint] = found
    return found


# A chat request is always bounded. Left unbounded, the server
# generates until the model emits its end token or the context
# fills — and a model that doesn't stop on its own (a base-style
# model in a chat template can ramble for thousands of tokens)
# keeps a CPU busy far past any reasonable wait while the
# dashboard shows "thinking…". That was the "she is not chatting"
# failure, reproduced against this bridge: no max_tokens rode
# along, the generation outran the old 120-second socket timeout,
# and the request died misreporting a healthy, answering server
# as "unreachable" — from the operator's chair, Navi simply never
# replied. max_tokens caps the work; the wait is sized to let a
# bounded reply finish even at slow CPU generation speeds.
CHAT_MAX_TOKENS = 1024
CHAT_HISTORY_MESSAGES = 24
CHAT_TIMEOUT = 600


def _chat_sanitize_messages(messages):
    # Build 6 (#224): the model server refuses a request carrying
    # ANY message with empty content (HTTP 400 "message must not
    # have empty content") — one hollow entry anywhere in the
    # history kills the whole answer. The field shape was a call:
    # a stored thread entry with no text (an exec-only or
    # placeholder row the dashboard's history builder passed
    # through) rode into the call's request and Navi's answer died
    # on it. The builder is the one funnel every chat request —
    # typed, called, or borrowed-back — passes through, so the
    # guard lives here: entries whose content is empty or
    # whitespace-only are dropped before the request is built.
    # A list with nothing left is a caller bug, and it is named
    # rather than sent on to be refused.
    out = []
    for m in messages or []:
        if not isinstance(m, dict):
            continue
        content = m.get("content")
        if content is None or not str(content).strip():
            continue
        out.append(m)
    if not out:
        raise RuntimeError(
            "the conversation held nothing the model could answer "
            "— every message in it was empty. Nothing was sent.")
    return out


# ---------------------------------------------------------------------
# The local prompt budget (v0.6.7e, #310).
#
# Until this build the local chat path measured NOTHING: the
# assembly appended block after block every turn (persona,
# Guardian, vitals, the intel brief, awareness, shell and case
# teaching, the Row, the case brief) and handed the result to a
# context the hardware profile had derived from RAM alone —
# 4096 on the 6–12 GiB class. On such a seat the standing
# prompt alone overflowed the window and EVERY local chat died
# at the model with out_of_context_due_to_long_message. The
# lend door has measured since Build 8 (#154) — refused whole,
# never truncated; the local path now keeps the same discipline
# with the opposite hand: it sheds BACKGROUND, lowest-value
# first, and never the operator's words. Three instruments:
#
#   * conditional teaching — the shell teaching rides only a
#     machine-directed turn (the #240 gate's own read), the
#     case teaching only a case-shaped one (a case bound or
#     open, or the turn naming case work);
#   * per-block caps — no single dynamic brief may eat the
#     window; an over-cap block is trimmed and the trim named;
#   * the budget itself — assembled prompt vs the seat's own
#     ctx_size (the lend estimator, shared), a reply's room
#     reserved (CHAT_MAX_TOKENS), background segments shed in
#     a fixed order, then per-turn system extras, then the
#     oldest history — and if even the standing core does not
#     fit, an honest named refusal instead of a dead model
#     call. The accounting rides the turn's payload as its
#     readout (promptBudget), so the size of her prompt is a
#     fact on the record, not a mystery in the field.
PROMPT_FLOOR_CTX = 4096
PROMPT_BLOCK_CAP_CHARS = 6000


def estimate_messages_tokens(messages):
    # The one estimator: four chars to a token plus a small
    # per-message frame — the lend door's arithmetic, shared.
    total = 0
    for m in messages or []:
        if isinstance(m, dict):
            total += len(str(m.get("content") or "")) // 4 + 4
    return total


def prompt_budget_apply(segments, rest, ctx_size):
    # segments: [{name, text, shed, cap}] in standing order —
    # shed is the shedding order (1 sheds first) or None for
    # the standing core, which never sheds. rest: the messages
    # after the system message; per-turn system extras in it
    # carry a "_budget_name" tag naming their drop order.
    # Returns (messages, report).
    try:
        ctx = int(ctx_size or 0)
    except Exception:
        ctx = 0
    if ctx <= 0:
        msgs = [{"role": "system",
                 "content": "\n\n".join(s["text"] for s in segments if s.get("text"))}] + list(rest)
        return msgs, {"estTokens": estimate_messages_tokens(msgs),
                      "ctxSize": ctx, "budgetTokens": 0, "shed": [],
                      "overflow": False}
    reserve = CHAT_MAX_TOKENS
    budget = ctx - reserve if ctx > reserve else ctx
    segs = [dict(s) for s in segments]
    shed = []
    for s in segs:
        if s.get("cap") and len(s.get("text") or "") > PROMPT_BLOCK_CAP_CHARS:
            s["text"] = (s["text"][:PROMPT_BLOCK_CAP_CHARS]
                         + "\n… (trimmed to this seat's prompt budget)")
            shed.append(s["name"] + " (trimmed)")

    def compose():
        return "\n\n".join(s["text"] for s in segs if s.get("text"))

    def total():
        return estimate_messages_tokens(
            [{"role": "system", "content": compose()}] + list(rest))

    est = total()
    if est > budget:
        for s in sorted((s for s in segs if s.get("shed") is not None),
                        key=lambda s: s["shed"]):
            if est <= budget:
                break
            if s.get("text"):
                s["text"] = ""
                shed.append(s["name"])
                est = total()
    if est > budget:
        for name in ("intel-seam", "rag", "shelf-ref"):
            while est > budget:
                kept, dropped = [], False
                for m in rest:
                    if not dropped and isinstance(m, dict) \
                            and m.get("_budget_name") == name:
                        dropped = True
                        shed.append(name)
                        continue
                    kept.append(m)
                if not dropped:
                    break
                rest = kept
                est = total()
    if est > budget:
        # History sheds oldest-first; the operator's current
        # turn (the last conversation message) never sheds.
        while est > budget:
            conv = [i for i, m in enumerate(rest)
                    if isinstance(m, dict) and m.get("role") != "system"]
            if len(conv) <= 1:
                break
            rest = rest[:conv[0]] + rest[conv[0] + 1:]
            shed.append("history")
            est = total()
    for m in rest:
        if isinstance(m, dict):
            m.pop("_budget_name", None)
    msgs = [{"role": "system", "content": compose()}] + list(rest)
    return msgs, {"estTokens": est, "ctxSize": ctx,
                  "budgetTokens": budget, "shed": shed,
                  "overflow": est > budget}


def llamafile_chat(messages, endpoint, model):
    # Guardian (Build 14, #156): every local generation is timed
    # and counted. The counter is what Guardian's is_generating
    # callback reads (pressure judges stricter mid-generation),
    # and the duration is what note_generation() learns slow-
    # generation collisions from. The return contract — the
    # reply text, or the same RuntimeErrors — is unchanged: this
    # wrapper only observes the call it wraps.
    messages = _chat_sanitize_messages(messages)
    _gen_t0 = time.time()
    with _GENERATION_LOCK:
        _GENERATION_ACTIVE["count"] += 1
    try:
        result = _llamafile_chat_impl(messages, endpoint, model)
    finally:
        with _GENERATION_LOCK:
            _GENERATION_ACTIVE["count"] = max(0, _GENERATION_ACTIVE["count"] - 1)
        try:
            GUARDIAN.note_generation(time.time() - _gen_t0)
        except Exception:
            pass
    # #187: the bridge's own stopwatch — the only numbers
    # about her speed that exist. Runs only on success (an
    # exception propagates past this point): a failed request
    # never replaces the last good measurement.
    try:
        _record_model_vitals(time.time() - _gen_t0)
    except Exception:
        pass
    return result


def _llamafile_chat_impl(messages, endpoint, model):
    # The llamafile server rejects a chat request whose JSON has no
    # "model" string (400 "JSON missing model string"), so a model
    # name rides along on every request — the id the server reports
    # for itself when it will say, otherwise the configured name.
    name = served_model(endpoint) or model
    body = json.dumps({
        "model": name,
        "messages": messages,
        "stream": False,
        "max_tokens": CHAT_MAX_TOKENS,
    }).encode()
    req = urllib.request.Request(
        endpoint.rstrip("/") + "/chat/completions",
        data=body,
        headers={"Content-Type": "application/json"},
    )
    try:
        with urllib.request.urlopen(req, timeout=CHAT_TIMEOUT) as resp:
            data = json.loads(resp.read().decode())
    except urllib.error.HTTPError as e:
        # Surface the server's own words — a bare "HTTP Error 400"
        # hides the reason it actually gave.
        try:
            detail = e.read().decode(errors="replace").strip()
        except Exception:
            detail = ""
        raise RuntimeError(
            "the llamafile server rejected the chat request (HTTP "
            + str(e.code) + ")" + ((": " + detail) if detail else "")
        )
    except TimeoutError:
        # The server is up — it was still generating when the wait
        # ran out. Saying "unreachable" here once sent the operator
        # hunting for a dead server that was never dead.
        raise RuntimeError(
            "navi.llamafile was still generating her answer after "
            + str(CHAT_TIMEOUT // 60) + " minutes, so the wait ran "
            "out — the server is up; the model is just slower than "
            "that on this machine right now. The request was "
            "bounded (" + str(CHAT_MAX_TOKENS) + " tokens at most), "
            "so sending the message again is safe."
        )
    except urllib.error.URLError as e:
        reason = getattr(e, "reason", e)
        if isinstance(reason, TimeoutError) or "timed out" in str(reason):
            raise RuntimeError(
                "navi.llamafile was still generating her answer "
                "when the wait ran out — the server is up; the "
                "model is just slower than the wait on this "
                "machine right now. Sending the message again "
                "is safe."
            )
        raise RuntimeError(
            "could not reach navi.llamafile at " + endpoint + ": "
            + str(reason)
        )
    # #187: the server's own account of the generation rides
    # home for the vitals record (llama.cpp servers report
    # timings — predicted_n/predicted_ms/prompt_ms — and the
    # OpenAI-shaped usage counts). Cleared as it is stashed,
    # so a response carrying neither never inherits the
    # previous request's figures.
    _LAST_GEN_SERVER.clear()
    _LAST_GEN_SERVER.update({
        "timings": data.get("timings"),
        "usage": data.get("usage"),
    })
    content = data["choices"][0]["message"]["content"]
    if not (content or "").strip():
        # An empty reply renders as silence in the thread — the
        # same "she is not chatting" symptom by another road. Name
        # it where the operator can see it instead.
        raise RuntimeError(
            "navi.llamafile answered with an empty reply — the "
            "model ended its turn without saying anything. Sending "
            "the message again usually gets a real answer."
        )
    return content


# --- Sync tombstones (0.2.0e Build 3, issues #176/#177) -------------
# A deletion on one machine must never resurrect from another
# machine's copy. Convergence (below) never deletes — it only
# merges — so a delete needs its own travelling record: the
# tombstone, {kind: {key: deletion epoch}}, kept in
# config/sync-state.json beside the store it describes and carried
# by every Quick Sync payload. The funnels that delete record
# here: chat_delete and the write-through's removals (chats),
# case_delete (cases), and the Vault's delete paths (notes).
# Stored backups are snapshots and are NEVER reached into — a
# tombstone governs convergence between live machines only; the
# copy inside an old backup stays in that backup (the #163 law).
# A tombstone yields to newer work: if the surviving copy carries
# a stamp LATER than the deletion, the copy wins and the tombstone
# is dropped — a thread genuinely continued after its deletion
# elsewhere is not a resurrection.
SYNC_STATE_PATH = os.path.join(ROOT, "config", "sync-state.json")
_SYNC_TOMBSTONE_KEEP_SECONDS = 90 * 86400
_sync_state_cache = {"mtime": None, "state": None}


def _sync_state_load():
    # Read-through cache on the file's mtime: the state is small,
    # written rarely, and read by hot paths (every chat delete).
    try:
        mtime = os.path.getmtime(SYNC_STATE_PATH)
    except Exception:
        mtime = None
    if _sync_state_cache["state"] is not None and _sync_state_cache["mtime"] == mtime:
        return _sync_state_cache["state"]
    state = {"tombstones": {"chats": {}, "cases": {}, "vault": {}, "daybook": {}},
             "lastSyncAt": None, "lastSyncEpoch": 0, "lastResult": "",
             "lastTrigger": "", "lastState": ""}
    try:
        with open(SYNC_STATE_PATH, encoding="utf-8") as f:
            raw = json.load(f)
        if isinstance(raw, dict):
            state.update({k: raw[k] for k in state if k in raw})
            tombs = raw.get("tombstones")
            if isinstance(tombs, dict):
                for kind in ("chats", "cases", "vault", "dossier", "daybook"):
                    if isinstance(tombs.get(kind), dict):
                        state["tombstones"][kind] = {
                            str(k): float(v) for k, v in tombs[kind].items()
                            if isinstance(v, (int, float))}
    except Exception:
        pass
    _sync_state_cache["mtime"] = mtime
    _sync_state_cache["state"] = state
    return state


def _sync_state_save(state):
    try:
        os.makedirs(os.path.dirname(SYNC_STATE_PATH), exist_ok=True)
        tmp = SYNC_STATE_PATH + ".tmp"
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(state, f, indent=2)
        os.replace(tmp, SYNC_STATE_PATH)
        try:
            _sync_state_cache["mtime"] = os.path.getmtime(SYNC_STATE_PATH)
        except Exception:
            _sync_state_cache["mtime"] = None
        _sync_state_cache["state"] = state
    except Exception:
        pass


def _sync_tombstones_pruned(tombs, now=None):
    now = now if now is not None else time.time()
    out = {}
    for kind in ("chats", "cases", "vault", "entries", "dossier", "daybook", "fedinets"):
        rows = tombs.get(kind) if isinstance(tombs, dict) else None
        out[kind] = {}
        if isinstance(rows, dict):
            for k, v in rows.items():
                if isinstance(v, (int, float)) and now - float(v) <= _SYNC_TOMBSTONE_KEEP_SECONDS:
                    out[kind][str(k)] = float(v)
    return out


def sync_record_tombstone(kind, key, at=None):
    # The deletion funnels' one call. Best-effort by design: a
    # tombstone that fails to record must never fail the delete
    # the operator asked for — the delete itself already happened.
    if kind not in ("chats", "cases", "vault", "entries", "dossier", "daybook", "fedinets") or not key:
        return
    try:
        state = _sync_state_load()
        tombs = _sync_tombstones_pruned(state.get("tombstones") or {})
        when = float(at) if at else time.time()
        prev = tombs[kind].get(str(key))
        if prev is None or when > prev:
            tombs[kind][str(key)] = when
        state["tombstones"] = tombs
        _sync_state_save(state)
        sync_note_write("tombstone")
    except Exception:
        pass


def _sync_merge_tombstones(local, remote):
    # Union by key, newest deletion wins — a tombstone one machine
    # has not seen yet must survive every round until it ages out,
    # or the delete it guards could resurrect from the one machine
    # that missed it.
    merged = _sync_tombstones_pruned(local or {})
    for kind, rows in _sync_tombstones_pruned(remote or {}).items():
        for k, v in rows.items():
            if k not in merged[kind] or v > merged[kind][k]:
                merged[kind][k] = v
    return merged


# --- Server-side chats (0.1.9e Build 12, issue #139) -----------------
# Chats lived only in browser localStorage (navi.chats) until this
# build: every install — and every browser origin — held its own
# set, and DHD into the server was continuous with the server only
# through the server's own browser. The store of record now lives
# HERE, in plain files under config/chats/ — one JSON file per chat
# (the Vault's plain-files rule), written atomically — so any door
# (the desktop window, a DHD session, a second browser) hydrates
# the same threads from the same install. localStorage is demoted
# to a cache and an offline fallback: the window writes through on
# every persist and hydrates back on load. Uploads follow chats:
# their index rides in config/chats/uploads.json beside the threads
# it belongs to (file bytes were always handed to the bridge
# inline; the index is the record).
# Convergence for foreign copies (a backup restore, #136's badge
# sync) is chats_converge: per chat, newest-wins by timestamp when
# one thread is the other's continuation — and when two copies of
# a thread have genuinely diverged, BOTH survive: the incoming one
# lands as a dated sibling thread, never a silent merge.

CHATS_DIR = os.path.join(ROOT, "config", "chats")

# v0.3.0e Build 1 (#200): the main chat is a named seat, not a
# deletable row. The page seeds it as id "1" ("Main session"); it
# may be EMPTIED (/clear or /cls) but never deleted — not from the
# page, not through the API, not by a wholesale replace, and never
# by a sync tombstone.
MAIN_CHAT_ID = "1"
CHATS_UPLOADS_PATH = os.path.join(CHATS_DIR, "uploads.json")


def _chat_clean_id(value):
    s = str(value if value is not None else "").strip()
    if not s or len(s) > 80:
        return None
    if not re.match(r"^[A-Za-z0-9][A-Za-z0-9._-]*$", s):
        return None
    if s in (".", "..", "uploads"):
        return None
    return s


def _chat_normalize(rec):
    if not isinstance(rec, dict):
        return None
    cid = _chat_clean_id(rec.get("id"))
    if cid is None:
        return None
    msgs = rec.get("messages")
    clean_msgs = [m for m in msgs if isinstance(m, dict)] if isinstance(msgs, list) else []
    out = {
        "id": cid,
        "title": str(rec.get("title") or "Chat")[:200],
        "messages": clean_msgs,
        "createdAt": rec.get("createdAt") if isinstance(rec.get("createdAt"), (int, float)) else 0,
        "updatedAt": rec.get("updatedAt") if isinstance(rec.get("updatedAt"), (int, float)) else 0,
    }
    # Case-tied chats (Build 15, #161): a chat bound to a case
    # carries the case slug in "case" — a reference only; the
    # case's content stays the Vault's plain files. The binding
    # survives normalization (and therefore write-through and
    # convergence) exactly like the thread itself does.
    case_ref = rec.get("case")
    if isinstance(case_ref, str) and re.match(r"^[a-z0-9][a-z0-9-]*$", case_ref):
        out["case"] = case_ref
    # Import provenance (v0.3.0e Build 5, #218): a chat minted by
    # convergence as another conversation's imported copy carries
    # that conversation's id in "fromSyncOf". It is the record's
    # sync identity — convergence, dedupe, and the tombstones all
    # key on it — so it survives normalization exactly like the
    # case binding does.
    origin_ref = rec.get("fromSyncOf")
    if isinstance(origin_ref, str) and _chat_clean_id(origin_ref):
        out["fromSyncOf"] = origin_ref
    # The reference shelf (v0.5.10e, #283): Wire cards pulled
    # into a case chat stand on the chat record itself, so the
    # shelf rides the chat store — and therefore Sync — exactly
    # like the thread does. It survives normalization (cleaned,
    # capped) exactly like the case binding does; an emptied
    # shelf leaves no field behind.
    shelf = rec.get("shelf")
    if isinstance(shelf, list):
        clean_shelf = _shelf_clean_list(shelf)
        if clean_shelf:
            out["shelf"] = clean_shelf
    # The clear epoch (v0.6.3e, #318): a main thread emptied by
    # /clear or /cls carries the moment it was emptied and the
    # seat that emptied it. The marker survives normalization —
    # and therefore write-through and convergence — exactly
    # like the case binding does; convergence honors it over
    # any content stamped at or before it (see chat_write).
    cleared = rec.get("clearedAt")
    if isinstance(cleared, (int, float)) and not isinstance(cleared, bool) \
            and cleared > 0:
        out["clearedAt"] = cleared
        by = rec.get("clearedBy")
        if isinstance(by, str) and by.strip():
            out["clearedBy"] = by.strip()[:80]
    return out


SHELF_TEXT_CAP = 20000
SHELF_MAX_REFS = 50


def _shelf_clean_list(shelf):
    out = []
    for ref in shelf:
        if not isinstance(ref, dict):
            continue
        rid = str(ref.get("id") or "")
        if not re.fullmatch(r"R\d+", rid):
            continue
        out.append({
            "id": rid,
            "tab": str(ref.get("tab") or "")[:20],
            "title": str(ref.get("title") or "")[:300],
            "link": str(ref.get("link") or "")[:1000],
            "source": str(ref.get("source") or "")[:120],
            "text": str(ref.get("text") or "")[:SHELF_TEXT_CAP],
            "pulledAt": ref.get("pulledAt") if isinstance(
                ref.get("pulledAt"), (int, float)) else 0,
            "pulledBy": "navi" if ref.get("pulledBy") == "navi" else "operator",
        })
        if len(out) >= SHELF_MAX_REFS:
            break
    return out


def _chat_path(cid):
    return os.path.join(CHATS_DIR, cid + ".json")


def _chat_clear_filter(rec):
    # The messages a record's clear epoch retires: lines
    # stamped at or before the moment the thread was emptied.
    # Post-clear lines stand; a line carrying no stamp is not
    # timestamped content and the epoch does not govern it.
    epoch = rec.get("clearedAt") if isinstance(rec, dict) else None
    msgs = (rec or {}).get("messages") or []
    if not isinstance(epoch, (int, float)) or isinstance(epoch, bool) \
            or not epoch:
        return list(msgs)
    return [m for m in msgs
            if not (isinstance(m, dict)
                    and isinstance(m.get("ts"), (int, float))
                    and not isinstance(m.get("ts"), bool)
                    and m["ts"] <= epoch)]


def _chat_seat_name():
    # This seat's name for a clear marker: the Mesh node name
    # when one is minted, the machine's hostname otherwise —
    # the same fall-back the Mesh identity itself uses.
    try:
        name = mesh_settings(load_config())["nodeName"]
        if name:
            return str(name)[:80]
    except Exception:
        pass
    return socket.gethostname()[:80]


def chat_write(rec):
    rec = _chat_normalize(rec)
    if rec is None:
        return None
    # v0.6.3e (#318): the clear epoch is enforced at the store
    # door, so NO write path — write-through, convergence, the
    # ghost fold, a restore — can land pre-clear content back
    # into an emptied main thread. The epoch only ever advances:
    # a stale window's copy, written without the marker (or with
    # an older one), adopts the stored epoch and is filtered by
    # it. When a write ADVANCES the epoch — a clear act landing
    # at this bridge — the marker takes this seat's name unless
    # the act already carries one.
    if _chat_is_main_family(rec):
        stored = None
        try:
            with open(_chat_path(rec["id"]), encoding="utf-8") as f:
                stored = _chat_normalize(json.load(f))
        except Exception:
            stored = None
        new_epoch = rec.get("clearedAt") or 0
        old_epoch = (stored or {}).get("clearedAt") or 0
        if old_epoch > new_epoch:
            rec["clearedAt"] = old_epoch
            if (stored or {}).get("clearedBy"):
                rec["clearedBy"] = stored["clearedBy"]
            new_epoch = old_epoch
        if new_epoch:
            if new_epoch > old_epoch and not rec.get("clearedBy"):
                rec["clearedBy"] = _chat_seat_name()
            rec["messages"] = _chat_clear_filter(rec)
    os.makedirs(CHATS_DIR, exist_ok=True)
    full = _chat_path(rec["id"])
    tmp = full + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(rec, f)
    os.replace(tmp, full)
    sync_note_write("chat")
    return rec


def chats_all():
    out = []
    if os.path.isdir(CHATS_DIR):
        for name in sorted(os.listdir(CHATS_DIR)):
            if not name.endswith(".json") or name == "uploads.json":
                continue
            try:
                with open(os.path.join(CHATS_DIR, name), encoding="utf-8") as f:
                    rec = json.load(f)
            except Exception:
                continue
            rec = _chat_normalize(rec)
            if rec is not None:
                out.append(rec)
    out.sort(key=lambda r: (r.get("updatedAt") or 0, str(r.get("id"))))
    return out


def chat_delete(cid, _record=True):
    cid = _chat_clean_id(cid)
    if cid is None:
        return False
    if cid == MAIN_CHAT_ID:
        return False
    rec = None
    try:
        with open(_chat_path(cid), encoding="utf-8") as f:
            rec = _chat_normalize(json.load(f))
    except Exception:
        rec = None
    try:
        os.remove(_chat_path(cid))
        # Build 3 (#177): the deletion is tombstoned, so a sync
        # with another machine's older copy cannot resurrect it.
        # Build 5 (#218): the tombstone is recorded for the
        # conversation as well as the id — an imported copy can
        # be re-derived under a different id by an older build's
        # convergence, and the conversation key is what refuses
        # its return. (_record=False is the sync apply path: the
        # merged tombstone set it applies already carries both,
        # and re-recording would creep the deletion time forward
        # on every seat that applies it.)
        if _record:
            sync_record_tombstone("chats", cid)
            if rec is not None:
                sync_record_tombstone("chats", _chat_conv_tomb_key(rec))
        return True
    except Exception:
        return False


def chats_uploads():
    try:
        with open(CHATS_UPLOADS_PATH, encoding="utf-8") as f:
            rows = json.load(f)
        if isinstance(rows, list):
            return [r for r in rows if isinstance(r, dict)]
    except Exception:
        pass
    return []


def chats_save_uploads(rows):
    os.makedirs(CHATS_DIR, exist_ok=True)
    tmp = CHATS_UPLOADS_PATH + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump([r for r in (rows or []) if isinstance(r, dict)], f)
    os.replace(tmp, CHATS_UPLOADS_PATH)


def chats_payload():
    return {"chats": chats_all(), "uploads": chats_uploads()}


def chats_replace_all(records, uploads):
    # The write-through path: the window that is actively writing
    # owns the store, and its word is authoritative — the whole set
    # lands as given, and threads it no longer holds are gone here
    # too (a chat deleted at one door stays deleted at every door).
    keep = set()
    written = 0
    # v0.4.10e (#253): a window still holding a stale copy of a
    # deleted thread cannot write it back through this door
    # either — tombstones govern every door. A record the sync
    # state names as deleted is refused (the main seat
    # excepted: it cannot be deleted, and its record is the
    # window's own). A genuinely new thread carries a fresh id
    # and creation stamp, so it is never covered by an old
    # conversation's tombstone.
    try:
        tombs = (_sync_state_load().get("tombstones") or {}).get("chats") or {}
    except Exception:
        tombs = {}
    for rec in (records or []):
        norm = _chat_normalize(rec)
        if norm is not None and tombs and not _chat_is_main_family(norm) and \
                (str(norm["id"]) in tombs or _chat_conv_tomb_key(norm) in tombs):
            continue
        saved = chat_write(rec)
        if saved is not None:
            keep.add(saved["id"])
            written += 1
    if os.path.isdir(CHATS_DIR):
        for name in sorted(os.listdir(CHATS_DIR)):
            if name.endswith(".json") and name != "uploads.json" and name[:-5] not in keep:
                if name[:-5] == MAIN_CHAT_ID:
                    # v0.3.0e Build 1 (#200): the main chat is
                    # never dropped by a wholesale replace, and
                    # never tombstoned.
                    continue
                dropped_rec = None
                try:
                    with open(os.path.join(CHATS_DIR, name), encoding="utf-8") as f:
                        dropped_rec = _chat_normalize(json.load(f))
                except Exception:
                    dropped_rec = None
                try:
                    os.remove(os.path.join(CHATS_DIR, name))
                    # Build 3 (#177): a thread the writing window
                    # dropped is a deletion like any other — it is
                    # tombstoned so sync cannot bring it back.
                    # Build 5 (#218): its conversation is tombstoned
                    # with it, for the same reason as chat_delete.
                    sync_record_tombstone("chats", name[:-5])
                    if dropped_rec is not None:
                        sync_record_tombstone("chats", _chat_conv_tomb_key(dropped_rec))
                except Exception:
                    pass
    if uploads is not None:
        chats_save_uploads(uploads)
    return {"ok": True, "chats": written}


def _chat_msgs_key(rec):
    return json.dumps(rec.get("messages") or [], sort_keys=True)


def _chat_msgs_prefix(shorter, longer):
    a = shorter.get("messages") or []
    b = longer.get("messages") or []
    if len(a) > len(b):
        return False
    return json.dumps(a, sort_keys=True) == json.dumps(b[:len(a)], sort_keys=True)


def _chat_msg_id(m):
    # A message's identity for merge dedupe (v0.4.10e, #251):
    # the canonical bytes of the message itself. Two seats
    # holding the same line hold the same identity, wherever
    # the thread around it has forked.
    return hashlib.sha256(json.dumps(m, sort_keys=True).encode()).hexdigest()


def _chat_merge_messages(local_msgs, remote_msgs):
    # The one-main merge (v0.4.10e, #251): the union of two
    # copies of the main thread, each message exactly once.
    # When every message on both sides carries a timestamp the
    # union is ordered by it (a stable sort, so equal stamps
    # keep their side's order); otherwise the local copy is the
    # spine and the foreign-only lines append in their own
    # order. Returns (messages, added_count).
    local_msgs = local_msgs or []
    remote_msgs = remote_msgs or []
    seen = {_chat_msg_id(m) for m in local_msgs}
    out = list(local_msgs)
    added = 0
    for m in remote_msgs:
        k = _chat_msg_id(m)
        if k not in seen:
            seen.add(k)
            out.append(m)
            added += 1
    if added and all(isinstance(m.get("ts"), (int, float))
                     for m in out if isinstance(m, dict)) and \
            all(isinstance(m, dict) for m in out):
        out = sorted(out, key=lambda m: m.get("ts") or 0)
    return out, added


def _chat_is_main_family(rec):
    # v0.4.10e (#251): the main thread is a SEAT, not one
    # conversation among many — there is exactly one, and every
    # record of it belongs to it: the seat itself (id "1") and
    # the sync-born copies older rounds parked as its siblings
    # (an import whose origin is the main seat). A case-bound
    # chat is never family, whatever its id looks like.
    if not isinstance(rec, dict) or rec.get("case"):
        return False
    if str(rec.get("id")) == MAIN_CHAT_ID:
        return True
    return _chat_origin(rec) == MAIN_CHAT_ID


def _chat_merge_verdict(cur, rec):
    # What merging `rec` into a store holding `cur` (same id) does:
    # "kept" (nothing to do), "replaced" (the incoming thread is
    # this thread continued, and newer — newest-wins), or "sibling"
    # (the two copies genuinely diverged — both survive).
    if _chat_msgs_key(cur) == _chat_msgs_key(rec):
        return "replaced" if (rec.get("updatedAt") or 0) > (cur.get("updatedAt") or 0) else "kept"
    if _chat_msgs_prefix(cur, rec) and (rec.get("updatedAt") or 0) >= (cur.get("updatedAt") or 0):
        return "replaced"
    if _chat_msgs_prefix(rec, cur):
        return "kept"
    return "sibling"


def _chat_origin(rec):
    # The conversation a record belongs to, for sync identity
    # (Build 5, #218): a minted import carries its origin in
    # fromSyncOf; a legacy minted sibling (Build 4 and earlier)
    # carries it in its id ("<origin>-sync-<stamp>"); anything
    # else is its own origin.
    o = rec.get("fromSyncOf")
    if isinstance(o, str) and _chat_clean_id(o):
        return o
    cid = str(rec.get("id") or "")
    if "-sync-" in cid:
        base = cid.split("-sync-", 1)[0]
        if _chat_clean_id(base):
            return base
    return cid


def _chat_conv_key(rec):
    # A conversation's identity across seats and rounds: its
    # origin id plus its birth stamp. Two seats' main chats are
    # both id "1" but were born at different moments — different
    # conversations, and this key is what can tell them apart.
    created = rec.get("createdAt")
    return (_chat_origin(rec), created if isinstance(created, (int, float)) else 0)


def _chat_conv_tomb_key(rec):
    origin, created = _chat_conv_key(rec)
    return "conv:" + origin + "|" + repr(created)


def _chat_sibling_id(rec):
    # The deterministic import identity (Build 5, #218): the
    # same foreign conversation derives the same sibling id on
    # every round and on every seat, so a tombstone, a dedupe,
    # and the next convergence all recognize it. Build 4 and
    # earlier minted a fresh timestamp id per import event —
    # an identity nothing could recognize twice, which is
    # exactly how deleted "(from sync)" copies kept coming
    # back under new names.
    origin, created = _chat_conv_key(rec)
    digest = hashlib.sha256((origin + "|" + repr(created)).encode()).hexdigest()[:10]
    return origin + "-sync-" + digest


def _chat_local_maps(rows):
    local = {str(r["id"]): r for r in rows}
    by_conv = {}
    for r in rows:
        by_conv.setdefault(_chat_conv_key(r), r)
    return local, by_conv


def _chats_dedupe_imports():
    # Build 5 (#218): a seat that ran Build 4 or earlier can
    # hold several minted copies of ONE imported conversation
    # (a fresh timestamp id per round — Alex's store held
    # three copies of one main session, two byte-identical).
    # Group minted imports by conversation identity; where the
    # copies form one lineage (each a prefix of the richest),
    # keep the richest and remove the rest, tombstoning the
    # removed ids so the other seat's copies of them are
    # refused at the door. Copies that genuinely diverged (no
    # prefix relation) both survive, as they always have.
    rows = chats_all()
    groups = {}
    for r in rows:
        if str(r["id"]) == MAIN_CHAT_ID:
            continue
        if "-sync-" not in str(r["id"]) and not r.get("fromSyncOf"):
            continue
        groups.setdefault(_chat_conv_key(r), []).append(r)
    removed = 0
    for group in groups.values():
        if len(group) < 2:
            continue
        group.sort(key=lambda r: (len(r.get("messages") or []), r.get("updatedAt") or 0))
        richest = group[-1]
        for r in group[:-1]:
            if _chat_msgs_prefix(r, richest):
                try:
                    os.remove(_chat_path(str(r["id"])))
                    sync_record_tombstone("chats", str(r["id"]))
                    removed += 1
                except Exception:
                    pass
    return removed

