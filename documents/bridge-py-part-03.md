# bridge.py — part 3 of 13

Source: `bridge.py` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 8439–12446 of bridge.py.
Top-level symbols in this part: fedichat_act, _fed_observe_members, federation_headline, federation_payload, federation_status_slice, federation_import_invitation_file, federation_act, fleet_settings, fleet_public, fleet_set_member, fleet_own_name, fleet_run_class, fleet_discover, fleet_split_clauses, fleet_classify, _fleet_named, fleet_plan, fleet_gap_sentence, fleet_send_contribution, fleet_record_text, fleet_compose, fleet_ask, fleet_chat_reply, _MeshFpError, _MeshDropped, _MeshUnreachable, _MeshTlsError, _MeshTimeout, _mesh_request, _mesh_send_prepared, _mesh_dial, _mesh_failure_stage, mesh_probe_peer, mesh_probe_tracked, mesh_probe_all_tracked, mesh_probe_peers_soon, mesh_health_loop, mesh_ensure_health_thread, _mesh_borrow_once, _mesh_relay_candidates, _mesh_borrow_via_relay, peer_fedinet_id, _peer_federation_made, _mesh_route_fields, _mesh_relay_fallback, _mesh_relay_probe, _mesh_try_borrow_inner, mesh_try_borrow, _chat_route_clean, _chat_routes_load, chat_route_get, _chat_route_store, chat_route_availability, chat_route_answer, chat_route_set, mesh_probe_reason, mesh_status_payload, _mesh_tls_presented, _mesh_bridge_processes, _mesh_listener_owner, _mesh_fp_short, _mesh_own_addresses, _mesh_is_own_door, mesh_health_check, _mesh_door_log, mesh_door_blocked, mesh_door_knock, mesh_door_success, mesh_door_drop, _MeshDoorServer, _mesh_sync_account_verdict, _mesh_sync_set_wire, _mesh_sync_decode, _mesh_sync_admit, mesh_handle_sync_hello, mesh_handle_sync_pull, mesh_handle_sync_exchange, MeshHandler, mesh_ensure_tunnel_key, mesh_tunnel_public_key.


def fedichat_act(data, via_dhd=False):
    # The dashboard door. Reading and posting are the
    # operator's own speech — NOT a crown act: via DHD they
    # ride the trusted-device session like every other
    # operator surface, with no Badge step-up. Build 2 grows
    # the door: rooms are created, revised and dissolved
    # here (the creator's acts — the bridge signs what the
    # creator's node does), media posts here, presence pings
    # here, the log is searched here, and a referral is filed
    # into a case here — through the case machinery's own
    # deterministic door, never by the referral itself.
    cfg = load_config()
    data = data or {}
    action = str(data.get("action") or "state")
    fed_id = str(data.get("fed") or "")
    room_id = str(data.get("room") or "")

    def fail(err):
        out = fedichat_payload(cfg, via_dhd)
        out["ok"] = False
        out["error"] = err
        return out

    if action == "post":
        msg, err = fedichat_post(cfg, fed_id, str(data.get("body") or ""),
                                 room_id, str(data.get("quote") or ""))
        if err:
            return fail(err)
        out = fedichat_payload(cfg, via_dhd)
        out["posted"] = msg["id"]
        return out
    if action == "media":
        msg, err = fedichat_media_post(
            cfg, fed_id, room_id, str(data.get("name") or ""),
            str(data.get("mime") or ""), str(data.get("dataB64") or ""))
        if err:
            return fail(err)
        out = fedichat_payload(cfg, via_dhd)
        out["posted"] = msg["id"]
        return out
    if action == "sync":
        fedinets = federation_settings(cfg)
        _idx, fed = _fed_find(fedinets, fed_id)
        if fed is not None:
            fedichat_sync_fed(cfg, fed)
        return fedichat_payload(cfg, via_dhd)
    if action == "settings":
        settings = fedichat_settings(cfg)
        retention = str(data.get("retention") or "")
        if retention in FEDICHAT_RETENTION_DAYS:
            settings["retention"] = retention
        if isinstance(data.get("relay"), bool):
            settings["relay"] = data["relay"]
        if fed_id and isinstance(data.get("muzzle"), bool):
            key = fed_id if not room_id else fed_id + "|" + room_id
            settings["muzzle"][key] = data["muzzle"]
        _fedichat_save_settings(cfg, settings)
        return fedichat_payload(cfg, via_dhd)
    if action in ("create-room", "create-dm", "set-members",
                  "dissolve", "tie-case"):
        fedinets = federation_settings(cfg)
        _idx, fed = _fed_find(fedinets, fed_id)
        if fed is None:
            return fail("No such fedinet on this node.")
        if action == "create-room":
            ttl = data.get("ttlSeconds")
            ttl = int(ttl) if isinstance(ttl, (int, float)) and int(ttl) > 0 else None
            d, err = fedichat_room_create(
                cfg, fed, str(data.get("name") or ""), "invite",
                data.get("members") or [], ttl,
                str(data.get("case") or ""))
        elif action == "create-dm":
            ms = mesh_settings(cfg)
            own = mesh_node_id() or ms["nodeId"]
            d, err = fedichat_room_create(
                cfg, fed, "", "dm",
                [own, str(data.get("member") or "")])
        elif action == "set-members":
            d, err = fedichat_room_revise(
                cfg, fed, room_id, members=data.get("members") or [])
        elif action == "dissolve":
            d, err = fedichat_room_revise(cfg, fed, room_id, dissolve=True)
        else:
            d, err = fedichat_room_revise(
                cfg, fed, room_id, case_slug=str(data.get("case") or ""))
        if err:
            return fail(err)
        out = fedichat_payload(cfg, via_dhd)
        out["roomId"] = d["room"]
        return out
    if action == "search":
        return fedichat_search(cfg, fed_id, room_id, str(data.get("q") or ""))
    if action == "presence":
        # The operator's own ephemeral state: held locally,
        # pinged to the room's members direct. It is never
        # written to any log — not here, not anywhere.
        fedinets = federation_settings(cfg)
        _idx, fed = _fed_find(fedinets, fed_id)
        if fed is None:
            return {"ok": False, "error": "No such fedinet on this node."}
        room = fedichat_room_find(fed, room_id)
        ms = mesh_settings(cfg)
        own = mesh_node_id() or ms["nodeId"]
        state = str(data.get("state") or "")[:40]
        if room is not None and own in fedichat_room_members(fed, room):
            _fedichat_presence_note(fed["id"], room_id, own, state)
            peers = {str(p.get("nodeId") or ""): p
                     for p in ms["borrow"]["peers"]}
            for mid in fedichat_room_members(fed, room):
                if mid == own:
                    continue
                peer = peers.get(mid)
                if peer is None:
                    continue
                try:
                    _fedichat_ask(peer, {
                        "action": "presence", "fedinetId": fed["id"],
                        "room": room_id, "state": state,
                        "requestId": secrets.token_hex(8)})
                except Exception:
                    pass
        return {"ok": True}
    if action == "file-referral":
        # The deterministic door, by the receiving operator's
        # act: a referral line becomes a case finding carrying
        # its title, its summary, and its custody — the room,
        # the line it derives from, its author. The referral
        # itself never files; this act is the filing.
        fedinets = federation_settings(cfg)
        _idx, fed = _fed_find(fedinets, fed_id)
        if fed is None:
            return fail("No such fedinet on this node.")
        room = fedichat_room_find(fed, room_id)
        if room is None:
            return fail("No such room in that fedinet.")
        slug = str(data.get("case") or "")
        head = None
        try:
            head = {h["slug"]: h
                    for h in cases_list_payload().get("cases") or []}.get(slug)
        except Exception:
            head = None
        if head is None:
            return fail("No such case on this node — a referral files into a case that stands here.")
        line = None
        for m in fedichat_log_read(fed["id"], room["room"]):
            if str(m.get("id") or "") == str(data.get("line") or ""):
                line = m
                break
        if line is None or line.get("kind") != "referral":
            return fail("That line is not a referral held in this room — nothing was filed.")
        op_names = _fedichat_op_names_fed(fed)
        author = (op_names.get(str(line.get("author") or ""))
                  or str(line.get("authorName") or ""))
        title = str((line.get("ref") or {}).get("title") or "Navi referral")
        body = ("Lead from a Navi-to-Navi exchange in "
                + fedichat_room_display(fed, room)
                + " (" + str(fed.get("name") or "") + ").\n\n"
                + title + "\n\n" + str(line.get("body") or "")
                + "\n\nCustody: referral line " + str(line.get("id") or "")
                + ", deriving from line " + str(line.get("replyTo") or "")
                + ", authored by Navi (" + author + "). Filed by the "
                "operator through the room's referral door.")
        res, ferr = case_file_entry(
            slug, "finding", "Referral: " + title, body,
            severity="info", author="operator")
        if res is None:
            return fail(str(ferr or "The filing did not land."))
        out = fedichat_payload(cfg, via_dhd)
        out["filed"] = str(res.get("path") or "filed")
        return out
    # state: opening a room asks for a round — gossip is how
    # the room stays current without a server.
    for fed in federation_settings(cfg):
        _fedichat_note_write(fed["id"])
    return fedichat_payload(cfg, via_dhd)


def _fed_observe_members(fed):
    # First sightings: a roster member this node has never seen
    # is stamped with the roster version at which it was first
    # seen — the fact the new-member tick reads. Existing stamps
    # never move.
    since = dict(fed.get("memberSince") or {})
    version = _fed_roster_version(fed)
    for m in _fed_roster_members(fed):
        if isinstance(m, dict):
            nid = str(m.get("nodeId") or "")
            if nid and nid not in since:
                since[nid] = version
    fed["memberSince"] = since
    return fed


def federation_headline(cfg):
    # The Mesh status headline, graduated (2026-11-13 ruling):
    # when this node keeps any fedinet, the headline is the
    # STATE — "Mesh Online" — not a census of pairs.
    return "Mesh Online" if federation_settings(cfg) else ""


def federation_payload(cfg, via_dhd=False):
    ms = mesh_settings(cfg)
    own_id = mesh_node_id() or ms["nodeId"]
    fedinets = federation_settings(cfg)
    out = []
    for fed in fedinets:
        members = []
        for m in _fed_roster_members(fed):
            if not isinstance(m, dict):
                continue
            nid = str(m.get("nodeId") or "")
            if nid == own_id:
                posture = "This node"
                if fed["role"] == "gateway":
                    posture += " — the gateway of this fedinet"
                if m.get("founding"):
                    posture += ", and its Founding Member"
                posture += "."
            elif nid in fed["untrusted"]:
                posture = ("Untrusted by you — never lent to, whatever the "
                           "roster or the ticks say.")
            elif nid in (fed.get("lendOverrides") or []):
                posture = "Ticked by you individually — this node lends to it."
            elif not fed["trustByDefault"]:
                posture = "Pairing only — trust-by-default is off for this fedinet."
            elif not _fed_roster_fresh(fed):
                posture = "Pairing only — the roster is past its freshness window."
            elif (fed.get("memberSince") or {}).get(nid, 0) > fed.get("trustBaselineVersion", 0) \
                    and not fed["lendToNew"]:
                posture = ("New since your trust baseline, and your new-member "
                           "tick is off — pairing only for now.")
            else:
                posture = "Auto-lent under trust-by-default."
            members.append({
                "n": m.get("n") or 0, "nodeId": nid,
                "name": str(m.get("name") or ""), "short": nid[:12],
                "posture": posture,
                "untrusted": nid in fed["untrusted"],
                "lendOverride": nid in (fed.get("lendOverrides") or []),
                "founding": bool(m.get("founding")),
                "gateway": nid == fed["gatewayNodeId"],
            })
        members.sort(key=lambda x: x["n"])
        entry = {
            "id": fed["id"], "name": fed["name"], "role": fed["role"],
            "gatewayNodeId": fed["gatewayNodeId"],
            "gatewayName": fed["gatewayName"],
            "gatewayShort": fed["gatewayNodeId"][:12],
            "rosterVersion": _fed_roster_version(fed),
            "rosterFresh": _fed_roster_fresh(fed),
            "rosterSyncedAt": fed.get("rosterSyncedAt") or 0,
            "staleSentence": "" if _fed_roster_fresh(fed) else _fed_roster_stale_sentence(fed),
            "trustByDefault": fed["trustByDefault"],
            "lendToNew": fed["lendToNew"],
            "borrowCap": fed.get("borrowCap") or 0,
            "members": members,
            "pendingJoins": [{"nodeId": str(p.get("nodeId") or ""),
                              "name": str(((p.get("card") or {}).get("name")) or ""),
                              "ts": p.get("ts") or 0}
                             for p in (fed.get("pendingJoins") or []) if isinstance(p, dict)],
            "ownNumber": next((m["n"] for m in members if m["nodeId"] == own_id), 0),
        }
        if fed["role"] == "gateway":
            # (v0.4.4e, #238): a gateway whose card cannot be
            # minted — no declared public endpoint — says so on
            # its own page, in the refusal's words, where the
            # card would have stood; a silent absence would read
            # as a rendering bug, not a missing declaration.
            refusal = gateway_card_refusal(cfg)
            if refusal:
                entry["gatewayCardPng"] = ""
                entry["gatewayCardRefusal"] = refusal
            else:
                try:
                    entry["gatewayCardPng"] = base64.b64encode(
                        build_gateway_card_png(cfg, fed)).decode("ascii")
                except Exception:
                    entry["gatewayCardPng"] = ""
        elif fed.get("gatewayCardPayload"):
            # A member re-shares the card it imported — the
            # gateway's own signed payload, re-drawn; nothing is
            # re-signed and nothing new is claimed.
            try:
                entry["gatewayCardPng"] = base64.b64encode(_mesh_card_png_render(
                    fed["gatewayCardPayload"], variant="gateway")).decode("ascii")
            except Exception:
                entry["gatewayCardPng"] = ""
        out.append(entry)
    return {
        "ok": True,
        "nodeId": own_id, "nodeName": ms["nodeName"],
        "viaDhd": bool(via_dhd),
        "badgeEnrolled": bool(_federation_badge_sha(cfg)),
        "invitationWaiting": os.path.isfile(FED_INVITATION_PATH),
        "knownDefault": {"name": ROCKPORT_FEDINET_NAME,
                         "note": ("Rockport Mesh is the known fedinet — its "
                                  "gateway card is the invitation. Import the "
                                  "card and this node is admitted; the roster "
                                  "itself lives on its gateway and is never "
                                  "baked into an install.")},
        "fedinets": out,
        # v0.6.1e (#293): this node's Fleet Answers posture —
        # the Federation page's toggle renders from this and
        # the act answers with it (the switch gate).
        "fleet": fleet_public(cfg),
        # v0.6.4e (#322): the Mesh's own state — the page's
        # head carries the Mesh toggle itself, on the grounds
        # that everything on this page rides on it: Mesh off
        # means fedinets, borrowing, FediChat and the fleet are
        # all dark, and the toggle that decides that belongs
        # where its consequences are administered. meshOn is
        # the stored setting; meshIdentity says whether this
        # node has its Mesh identity minted.
        "meshOn": bool(ms.get("enabled")),
        "meshIdentity": bool(own_id),
        "audit": federation_audit_read(),
    }


def federation_status_slice(cfg):
    # v0.3.0e Build 2 (#204): the status column's FediNet
    # sections, as data — one entry per membership, its state
    # the federation machinery's own verdict, never a cosmetic
    # guess:
    #   ok      — the roster is inside its freshness window and
    #             the gateway road answers (this node IS the
    #             gateway, or its probes of the gateway hold a
    #             success).
    #   stale   — the roster is past the 72-hour freshness
    #             window: the machinery's own pairing-only
    #             degradation, a named middle state.
    #   down    — the gateway's probes hold a recorded failure:
    #             the fedinet cannot be reached right now.
    #   unknown — no probe of the gateway has completed yet.
    # No memberships → the empty list; the column itself
    # prints the decree's "You are not federated." line.
    out = []
    for fed in federation_settings(cfg):
        if not _fed_roster_fresh(fed):
            state = "stale"
        elif fed["role"] == "gateway":
            state = "ok"
        else:
            cache = MESH_STATE["probe_cache"].get(fed["gatewayNodeId"])
            if cache is None:
                state = "unknown"
            elif cache[1] is None:
                state = "down"
            else:
                state = "ok"
        out.append({"name": fed["name"], "state": state})
    return out


def federation_import_invitation_file():
    # The preload story, as a file contract: a gateway card
    # waiting at config/federation/invitation.png — placed there
    # by a container image's first boot, or by an operator's hand
    # — is imported ADMISSION-ONLY at bridge start, once, and
    # renamed aside so it never imports twice. Container first
    # boots carry the ruled Rockport default (trust-by-default
    # ON for the imported fedinet); a hand-dropped file on bare
    # metal imports with trust OFF — enabling trust stays the
    # operator's crown act either way, and nothing lends until a
    # signed roster syncs.
    if not os.path.isfile(FED_INVITATION_PATH):
        return ""
    try:
        with open(FED_INVITATION_PATH, "rb") as f:
            data = f.read()
    except Exception:
        return ""
    payload, err = parse_gateway_card_png(data)
    aside = os.path.join(FED_DIR, "invitation-refused.png")
    if payload is None:
        try:
            os.replace(FED_INVITATION_PATH, aside)
        except Exception:
            pass
        federation_audit("invitation", "", "The waiting invitation was refused: " + err, "desk")
        return err
    cfg = load_config()
    trust_default = os.environ.get("APEX_CONTAINER") == "1"
    fed, err = _fed_import_gateway_payload(cfg, payload, trust_default=trust_default)
    try:
        os.replace(FED_INVITATION_PATH,
                   os.path.join(FED_DIR, "invitation-imported.png"))
    except Exception:
        pass
    if fed is None:
        federation_audit("invitation", "", "The waiting invitation was refused: " + err, "desk")
        return err
    federation_audit("invitation", fed["name"],
                     "A waiting gateway card was imported at start — admission "
                     "and introduction only"
                     + ("; the container default set trust-by-default ON for "
                        "this fedinet (it engages at the first signed roster "
                        "sync)." if trust_default else "."),
                     "desk")
    return fed["name"]


def federation_act(data, via_dhd=False):
    # POST /api/federation — every Federation-page act, tiered
    # per the DHD ruling. Read + revoke ride any door; manage
    # acts through DHD need the Badge step-up; crown acts need
    # the ceremony everywhere (Badge added by the door). Every
    # act answers with the fresh state, so the page, the stored
    # config, and a reload read one truth (the switch gate).
    if not isinstance(data, dict):
        return {"ok": False, "error": "That request was not shaped like a Federation act."}
    act = str(data.get("act") or "")
    cfg = load_config()
    fedinets = federation_settings(cfg)
    via = "dhd" if via_dhd else "desk"

    def state(**extra):
        out = federation_payload(load_config(), via_dhd)
        out.update(extra)
        return out

    def refuse(sentence):
        out = state()
        out["ok"] = False
        out["error"] = sentence
        return out

    if act == "state":
        return state()

    if act in ("badge-enroll", "badge-clear"):
        if via_dhd:
            return refuse("The Badge step-up is enrolled in person, at the "
                          "machine itself — the factor that guards the DHD "
                          "door is never set through the DHD door.")
        if act == "badge-enroll":
            raw = _fed_badge_bytes(data)
            if raw is None:
                return refuse("No Badge picture rode that request — enrollment "
                              "needs the Operator Badge image itself.")
            rawcfg = dict(_federation_slice(cfg))
            rawcfg["badgeSha256"] = hashlib.sha256(raw).hexdigest()
            cfg["federation"] = rawcfg
            save_config(cfg)
            federation_audit("badge-enroll", "", "The Operator Badge was "
                             "enrolled as the DHD step-up factor (fingerprint "
                             "only; the picture never stays on this node).", via)
            return state(sentence="The Operator Badge is enrolled — crown and "
                                  "manage acts can now ride the DHD door under "
                                  "its step-up.")
        rawcfg = dict(_federation_slice(cfg))
        rawcfg.pop("badgeSha256", None)
        cfg["federation"] = rawcfg
        save_config(cfg)
        federation_audit("badge-clear", "", "The Operator Badge enrollment "
                         "was cleared — DHD crown and manage acts are shut "
                         "until a Badge is enrolled again.", via)
        return state(sentence="The Badge enrollment is cleared.")

    if act == "fleet":
        # v0.6.1e (#293): Fleet Answers membership — this node's
        # own posture, so it is a plain act, not a crown act and
        # not a manage act: joining grants nobody anything (every
        # contribution is still judged at the lender's own door,
        # under its caps and consent), and leaving must never be
        # gated — an operator can always take their machine out.
        # Joining is refused BY NAME when there is no fleet to
        # join: the Mesh off, or this node's identity unminted —
        # the toggle stays off and the card carries the sentence.
        want = data.get("on") is True
        if want and not fleet_settings(cfg)["member"]:
            ms = mesh_settings(cfg)
            if not ms["enabled"] or not mesh_node_id():
                return refuse("Fleet Answers rides the Mesh \u2014 this "
                              "node's Mesh is off, or its identity is not "
                              "minted yet, so there is no fleet to join. "
                              "The toggle stays off.")
        if want != fleet_settings(cfg)["member"]:
            fleet_set_member(want, load_config())
            federation_audit("fleet-join" if want else "fleet-leave", "",
                             "this node " + ("joined" if want else "left")
                             + " the fleet (Federation page)", via)
        return state(sentence=("This node is in the fleet \u2014 it answers "
                               "fleet asks from paired machines and joined "
                               "fedinets, under the same caps as borrowing.")
                       if want else
                       ("This node has left the fleet \u2014 it serves no "
                        "fleet asks, and its own asks run locally."))
    if act == "found":
        name = str(data.get("name") or "").strip()[:80]
        if not name:
            return refuse("A fedinet needs a name — the name rides its "
                          "gateway card's face and its roster's head.")
        if any(f["name"].lower() == name.lower() for f in fedinets):
            return refuse("This node already keeps a fedinet named '" + name + "'.")
        ok, err = _fed_crown_ok(cfg, via_dhd, data, _fed_confirm_found(name))
        if not ok:
            return refuse(err)
        ms = mesh_settings(cfg)
        own_id = mesh_node_id() or ms["nodeId"]
        if not own_id:
            return refuse("This node's Mesh identity is not minted yet — "
                          "enable the Mesh and let it mint its keypair first.")
        own = mesh_card(cfg)
        fed = _fed_sanitize({
            "name": name, "role": "gateway",
            "gatewayNodeId": own_id, "gatewayName": own.get("name"),
            "gatewayCard": own, "foundedAt": int(time.time()),
            "memberSince": {own_id: 1},
        })
        entry = _fed_member_entry(cfg, own, 1, int(time.time()))
        entry["founding"] = True
        fed["roster"] = {"version": 0, "members": [entry]}
        fedinets.append(fed)
        fed = _fed_gateway_publish(cfg, fedinets, fed)
        federation_audit("found", name, "Founded the fedinet and signed "
                         "roster v1 — this node is its gateway and its "
                         "Founding Member (Node 1).", via)
        return state(sentence="'" + name + "' is founded — this node is its "
                              "gateway and its Founding Member. Its gateway "
                              "card is on this page, ready to hand out.")

    # Everything below names a fedinet.
    fed_id = str(data.get("fedinetId") or "")
    idx, fed = _fed_find(fedinets, fed_id)
    if fed is None and act not in ("import", "import-invitation"):
        return refuse("That fedinet is not on this node.")

    if act in ("import", "import-invitation"):
        ok, err = _fed_badge_ok(cfg, via_dhd, data)
        if not ok:
            return refuse(err)
        if act == "import-invitation":
            if not os.path.isfile(FED_INVITATION_PATH):
                return refuse("No invitation is waiting on this node.")
            federation_import_invitation_file()
            return state(sentence="The waiting invitation was imported — "
                                  "admission and introduction only.")
        raw = data.get("cardPngB64") or ""
        try:
            png = base64.b64decode(str(raw), validate=True)
        except Exception:
            return refuse("That file did not arrive whole — import the "
                          "gateway card's PNG file itself.")
        payload, perr = parse_gateway_card_png(png)
        if payload is None:
            return refuse(perr)
        fed2, ierr = _fed_import_gateway_payload(load_config(), payload)
        if fed2 is None:
            return refuse(ierr)
        # v0.6.4e (#321): a manual import overrules a recorded
        # departure — the tombstone clears at THIS door only.
        _fed_clear_departure(fed2["id"])
        federation_audit("import", fed2["name"], "Imported the gateway card "
                         "of '" + fed2["name"] + "' — admission and "
                         "introduction only; no trust was granted by the "
                         "import.", via)
        return state(sentence="'" + fed2["name"] + "' admitted this node — "
                              "its gateway is paired as an introduction. "
                              "Trust is a separate act, below.")

    if act == "gateway-card":
        if fed["role"] != "gateway":
            return refuse("Only the gateway mints its fedinet's gateway card.")
        refusal = gateway_card_refusal(cfg)
        if refusal:
            return refuse(refusal)
        try:
            png = build_gateway_card_png(cfg, fed)
        except Exception as e:
            return refuse("The gateway card could not be drawn: " + str(e)[:120])
        return state(cardPngB64=base64.b64encode(png).decode("ascii"),
                     sentence="The gateway card for '" + fed["name"] + "' — "
                              "green, and it admits whoever imports it to an "
                              "introduction, nothing more.")

    if act == "knock":
        ms = mesh_settings(cfg)
        gw = next((p for p in ms["borrow"]["peers"]
                   if p["nodeId"] == fed["gatewayNodeId"]), None)
        if gw is None:
            return refuse("The gateway is not paired on this node — import "
                          "its gateway card first.")
        payload = {"action": "join", "fedinetId": fed["id"],
                   "card": mesh_card(cfg), "requestId": secrets.token_hex(8)}
        body = json.dumps(payload).encode()
        try:
            _code, raw = _mesh_request(gw, "POST", "/mesh/roster", body,
                                       payload["requestId"], 15)
            ans = json.loads(raw.decode("utf-8", "replace"))
        except Exception as e:
            # Build 6 (#226): the failure names its stage and the
            # road it died on, and the FAILED knock is audited —
            # one entry per operator-initiated attempt, the same
            # cadence a failed sync already kept. (Before this, a
            # failed knock left no trace anywhere, and a live
            # diagnosis misread the silent log as "no knock was
            # ever sent".)
            stage = _mesh_failure_stage(e)
            federation_audit("knock", fed["name"],
                             "The knock did not reach the gateway — "
                             + (stage or "the request failed before "
                                "any answer") + ".", via)
            if stage:
                return refuse("The gateway did not answer the knock — "
                              + stage + ". The road is dark, or the "
                              "gateway is off.")
            return refuse("The gateway did not answer the knock — the road "
                          "is dark, or the gateway is off.")
        if not isinstance(ans, dict) or not ans.get("ok"):
            return refuse("The gateway refused the knock (" +
                          str((ans or {}).get("reason") or "refused") + ").")
        if ans.get("roster"):
            fedinets2 = federation_settings(load_config())
            _i, cur = _fed_find(fedinets2, fed["id"])
            _f, aerr = _fed_accept_roster(load_config(), fedinets2, cur, ans["roster"])
            if not aerr:
                _fed_observe_members(_f)
                fedinets3 = federation_settings(load_config())
                _i3, cur3 = _fed_find(fedinets3, fed["id"])
                cur3["memberSince"] = _f["memberSince"]
                fedinets3[_i3] = cur3
                _federation_save(load_config(), fedinets3)
                federation_apply(load_config())
        federation_audit("knock", fed["name"], "Knocked on the gateway's "
                         "door — admission is the gateway operator's act.",
                         via)
        return state(sentence="The knock is sent — this node waits in the "
                              "gateway's pending list until its operator "
                              "admits it.")

    if act == "sync":
        if fed["role"] == "gateway":
            fed = _fed_gateway_publish(cfg, fedinets, fed)
            return state(sentence="This node IS the gateway — its roster is "
                                  "the signed word, republished at version " +
                                  str(_fed_roster_version(fed)) + ".")
        sentence = _fed_roster_sync(load_config(), fed)
        fedinets2 = federation_settings(load_config())
        _i, cur = _fed_find(fedinets2, fed["id"])
        if cur is not None:
            _fed_observe_members(cur)
            fedinets2[_i] = cur
            _federation_save(load_config(), fedinets2)
        federation_audit("sync", fed["name"], sentence, via)
        return state(sentence=sentence)

    if act in ("admit", "deny", "expel"):
        if fed["role"] != "gateway":
            return refuse("Only the gateway admits, denies, or expels — "
                          "this node is a member of '" + fed["name"] + "'.")
        ok, err = _fed_badge_ok(cfg, via_dhd, data)
        if not ok:
            return refuse(err)
        nid = str(data.get("nodeId") or "")
        if act == "deny":
            fed = dict(fed)
            fed["pendingJoins"] = [p for p in (fed.get("pendingJoins") or [])
                                   if str(p.get("nodeId") or "") != nid]
            fedinets[idx] = fed
            _federation_save(cfg, fedinets)
            federation_audit("deny", fed["name"], "Denied the pending knock "
                             "from node " + nid[:12] + ".", via)
            return state(sentence="The knock was denied.")
        joiner = next((p for p in (fed.get("pendingJoins") or [])
                       if str(p.get("nodeId") or "") == nid), None)
        if act == "admit":
            if joiner is None:
                return refuse("That knock is no longer pending.")
            peer, perr = mesh_parse_card(json.dumps(joiner.get("card") or {}))
            if peer is None:
                return refuse("The pending card no longer validates: " + (perr or ""))
            cfg2 = load_config()
            ms2 = mesh_settings(cfg2)
            existing = next((p for p in ms2["borrow"]["peers"]
                             if p["nodeId"] == peer["nodeId"]), None)
            if existing is not None:
                # Already paired here (by hand, before the knock):
                # the record keeps its operator facts — relay
                # ticks, the tunnel route — and gains only its
                # federation provenance.
                block = dict(cfg2.get("mesh") or {})
                borrow = dict(block.get("borrow") or {})
                new_peers = []
                for p in borrow.get("peers") or []:
                    if isinstance(p, dict) and str(p.get("nodeId") or "") == peer["nodeId"]:
                        p = dict(p)
                        p["viaFedinet"] = fed["id"]
                        # autoPaired stays as it was (False): the
                        # federation marked this record, it did not
                        # make it — an expulsion unmarks it again
                        # rather than deleting the pairing.
                    new_peers.append(p)
                borrow["peers"] = new_peers
                block["borrow"] = borrow
                cfg2["mesh"] = block
                save_config(cfg2)
            else:
                peer["viaFedinet"] = fed["id"]
                peer["autoPaired"] = True
                mesh_add_peer(cfg2, peer)
            fed = dict(fed)
            fed["pendingJoins"] = [p for p in (fed.get("pendingJoins") or [])
                                   if str(p.get("nodeId") or "") != nid]
            members = _fed_roster_members(fed)
            nxt = max([int(m.get("n") or 0) for m in members if isinstance(m, dict)] + [0]) + 1
            fed["roster"] = dict(fed.get("roster") or {})
            fed["roster"]["members"] = members + [
                _fed_member_entry(cfg2, peer, nxt, int(time.time()))]
            fedinets[idx] = fed
            # No save here: the publish reconciles this intent
            # against the last SIGNED roster on disk and saves
            # the whole slice (pending list included) itself.
            fed = _fed_gateway_publish(load_config(), federation_settings(load_config()), fed)
            _fed_observe_members(fed)
            fedinets4 = federation_settings(load_config())
            _i4, cur4 = _fed_find(fedinets4, fed["id"])
            cur4["memberSince"] = fed["memberSince"]
            fedinets4[_i4] = cur4
            _federation_save(load_config(), fedinets4)
            federation_audit("admit", fed["name"], "Admitted " +
                             str(peer.get("name") or nid[:12]) + " as Node " +
                             str(nxt) + " of the fedinet; roster republished.",
                             via)
            return state(sentence=str(peer.get("name") or "The node") +
                                  " is Node " + str(nxt) + " of '" + fed["name"] +
                                  "' — the signed roster carries it now.")
        # expel
        members = [m for m in _fed_roster_members(fed)
                   if str((m or {}).get("nodeId") or "") != nid]
        if len(members) == len(_fed_roster_members(fed)):
            return refuse("That node is not on the roster.")
        fed = dict(fed)
        fed["roster"] = dict(fed.get("roster") or {})
        fed["roster"]["members"] = members
        fedinets[idx] = fed
        fed = _fed_gateway_publish(load_config(), federation_settings(load_config()), fed)
        federation_audit("expel", fed["name"], "Expelled node " + nid[:12] +
                         " — a signed roster revision carries it at the "
                         "next sync; hand-made pairings are untouched.", via)
        return state(sentence="Node " + nid[:12] + " is off the roster — the "
                              "expulsion rides the next signed revision.")

    if act == "leave":
        if fed["role"] == "gateway":
            return refuse("A gateway cannot leave its own fedinet — it "
                          "dissolves only when its operator retires the "
                          "gateway itself.")
        ms = mesh_settings(cfg)
        keep = [p for p in ms["borrow"]["peers"] if p.get("viaFedinet") != fed["id"]]
        if len(keep) != len(ms["borrow"]["peers"]):
            cfg2 = load_config()
            block = dict(cfg2.get("mesh") or {})
            borrow = dict(block.get("borrow") or {})
            borrow["peers"] = keep
            block["borrow"] = borrow
            cfg2["mesh"] = block
            save_config(cfg2)
        fedinets.pop(idx)
        _federation_save(load_config(), fedinets)
        # v0.6.4e (#321): the departure is tombstoned — without
        # it, the fedinet's gateway card rides the ChipDeck
        # forever, and any seat that converges the deck
        # re-admits what the operator left (the ghost fedinet).
        sync_record_tombstone("fedinets", fed["id"])
        federation_audit("leave", fed["name"], "Left the fedinet — its "
                         "federation-made pairings went with it; hand-made "
                         "pairings stand.", via)
        return state(sentence="This node has left '" + fed["name"] + "'.")

    if act == "dissolve":
        # v0.6.4e (#321): a gateway retires its own fedinet.
        # Until this build the door did not exist — leave
        # refused a gateway outright and nothing else could
        # end a fedinet, so a gateway's fedinet could only
        # ghost. Dissolving is a crown act: the spoken confirm
        # (louder when the roster holds other members — it
        # names the stranding), the audit, and the departure
        # tombstone, so no deck on any seat re-deals or
        # re-admits what was dissolved.
        if fed["role"] != "gateway":
            return refuse("Only the gateway dissolves its fedinet — a "
                          "member leaves, above.")
        ok, err = _fed_crown_ok(cfg, via_dhd, data, _fed_confirm_dissolve(fed))
        if not ok:
            return refuse(err)
        ms = mesh_settings(cfg)
        keep = [p for p in ms["borrow"]["peers"] if p.get("viaFedinet") != fed["id"]]
        if len(keep) != len(ms["borrow"]["peers"]):
            cfg2 = load_config()
            block = dict(cfg2.get("mesh") or {})
            borrow = dict(block.get("borrow") or {})
            borrow["peers"] = keep
            block["borrow"] = borrow
            cfg2["mesh"] = block
            save_config(cfg2)
        fedinets.pop(idx)
        _federation_save(load_config(), fedinets)
        sync_record_tombstone("fedinets", fed["id"])
        federation_audit("dissolve", fed["name"], "Dissolved the fedinet — "
                         "its federation-made pairings went with it; "
                         "hand-made pairings stand. The departure is "
                         "tombstoned, so the deck cannot re-deal it.", via)
        return state(sentence="'" + fed["name"] + "' is dissolved. Its "
                              "members keep their own pairings; the "
                              "fedinet itself is gone, and the deck will "
                              "not deal it again.")

    if act == "trust":
        want = data.get("on") is True
        if want:
            ok, err = _fed_crown_ok(cfg, via_dhd, data, _fed_confirm_trust(fed["name"]))
            if not ok:
                return refuse(err)
            fed = dict(fed)
            fed["trustByDefault"] = True
            version = _fed_roster_version(fed)
            if not fed.get("trustBaselineVersion"):
                fed["trustBaselineVersion"] = version
            _fed_observe_members(fed)
            fedinets[idx] = fed
            _federation_save(cfg, fedinets)
            federation_apply(load_config())
            federation_audit("trust", fed["name"], "Enabled DEEP trust — "
                             "the signed roster now auto-pairs and auto-lends "
                             "on this node (trust baseline: roster version " +
                             str(fed["trustBaselineVersion"]) + ").", via)
            return state(sentence="Deep trust is ON for '" + fed["name"] +
                                  "' — its signed roster auto-pairs and "
                                  "auto-lends on this node.")
        fed = dict(fed)
        fed["trustByDefault"] = False
        fedinets[idx] = fed
        _federation_save(cfg, fedinets)
        federation_audit("trust", fed["name"], "Turned trust-by-default OFF "
                         "— auto-lending ends now; pairings already made "
                         "stay pairings.", via)
        return state(sentence="Trust-by-default is OFF for '" + fed["name"] +
                              "' — auto-lending ends; pairings stay.")

    if act == "tick":
        ok, err = _fed_badge_ok(cfg, via_dhd, data)
        if not ok:
            return refuse(err)
        fed = dict(fed)
        fed["lendToNew"] = data.get("on") is True
        fedinets[idx] = fed
        _federation_save(cfg, fedinets)
        federation_audit("tick", fed["name"], "Set 'lend to new members "
                         "automatically' " + ("ON." if fed["lendToNew"] else
                         "OFF — new members are pairing-only for this node "
                         "until ticked."), via)
        return state(sentence="The new-member tick is " +
                              ("ON." if fed["lendToNew"] else "OFF."))

    if act == "untrust":
        nid = str(data.get("nodeId") or "")
        want = data.get("on") is True
        if not want:
            ok, err = _fed_badge_ok(cfg, via_dhd, data)
            if not ok:
                return refuse(err)
        fed = dict(fed)
        untrusted = [u for u in fed["untrusted"] if u != nid]
        if want:
            untrusted.append(nid)
        fed["untrusted"] = untrusted
        fedinets[idx] = fed
        _federation_save(cfg, fedinets)
        federation_audit("untrust" if want else "retrust", fed["name"],
                         ("Untrusted node " + nid[:12] + " — it is never "
                          "lent to, whatever the roster or ticks say.")
                         if want else
                         ("Restored node " + nid[:12] + " to the fedinet's "
                          "ordinary posture."), via)
        return state(sentence="Node " + nid[:12] + " is " +
                              ("untrusted." if want else "back under the "
                               "fedinet's ordinary posture."))

    if act == "lend":
        ok, err = _fed_badge_ok(cfg, via_dhd, data)
        if not ok:
            return refuse(err)
        nid = str(data.get("nodeId") or "")
        want = data.get("on") is True
        fed = dict(fed)
        overrides = [u for u in (fed.get("lendOverrides") or []) if u != nid]
        if want:
            overrides.append(nid)
        fed["lendOverrides"] = overrides
        fedinets[idx] = fed
        _federation_save(cfg, fedinets)
        federation_audit("lend", fed["name"], "Ticked node " + nid[:12] +
                         " individually — this node lends to it."
                         if want else
                         "Removed the individual lend tick for node " +
                         nid[:12] + ".", via)
        return state(sentence="The individual lend tick for node " + nid[:12] +
                              " is " + ("ON." if want else "OFF."))

    if act == "cap":
        ok, err = _fed_badge_ok(cfg, via_dhd, data)
        if not ok:
            return refuse(err)
        fed = dict(fed)
        fed["borrowCap"] = max(0, min(100000, _mesh_int(data.get("cap"), 0)))
        fedinets[idx] = fed
        _federation_save(cfg, fedinets)
        federation_audit("cap", fed["name"], "Set the fedinet borrow cap to " +
                         (str(fed["borrowCap"]) + " generations an hour."
                          if fed["borrowCap"] else "no cap."), via)
        return state(sentence="The borrow cap for '" + fed["name"] + "' is " +
                              (str(fed["borrowCap"]) + " generations an hour."
                               if fed["borrowCap"] else "off."))

    return refuse("That Federation act is not one this build knows.")


# -- Fleet Answers (v0.6.1e, #293) -------------------------------
# One ask, divided into WHOLE sub-tasks — each answered whole
# by the machine best placed for it, composed into one answer
# with the seams showing: every contribution signed with its
# machine's name. Design of record:
# files/fleet-answers-prep.md, with Alex's 2026-11-18 rulings:
# membership is a toggle on the Federation page, OFF by
# default, join/leave at will; the switch gate holds; a node
# leaving mid-ask degrades the answer by the named-gap rule.
# The doctrine locks, restated where they bind:
#   * The unit of work stays WHOLE (Mesh Doctrine II) — a
#     fleet ask fans out into whole sub-tasks in the existing
#     classes (generate / collect / lookup / report); tokens
#     and layers are never split, under any name.
#   * peek NEVER fans out — a peek-named piece is held back
#     and named in the plan, never sent anywhere.
#   * One hop, hard — a contribution never fans out again:
#     the borrow door refuses a fleet block that arrives
#     already inside a fleet ask (its depth), and the ask
#     machinery refuses to run from inside a contribution.
#   * No new trust — contributions ride the existing borrow
#     door, pairings and roster terms exactly as they stand,
#     and count against the lender's existing caps, one
#     ledger entry per contribution, cross-referenced to the
#     fleet ask's id on both sides of the door.
#   * A dark, refusing, timed-out, preempted or departing
#     contributor degrades the answer with the gap NAMED —
#     never a silently thinner answer, never a hang: the ask
#     carries a deadline and ships a badged partial.
#   * The fleet route never gates a plain answer (#289): the
#     judged route fires only on a multi-part ask, and a plain
#     single-part question falls through to the ordinary path
#     untouched.
FLEET_CLASSES = ("generate", "collect", "lookup", "report")
FLEET_CLASSES_ALL = FLEET_CLASSES + ("peek",)
FLEET_STATE = {"member_stamp": 0, "executing": 0}
_FLEET_FRAMING = {
    "generate": ("You are answering one whole sub-task of a larger ask, "
                 "as one machine in a fleet. Answer this piece whole and "
                 "on its own terms; do not mention the fleet."),
    "lookup": ("You are answering one lookup of a larger ask, as one "
               "machine in a fleet. Answer directly and briefly, from "
               "what you know; name uncertainty as uncertainty; do not "
               "invent specifics."),
}
_FLEET_TRIGGER = re.compile(
    r"^\s*(?:fleet\s*:\s*|ask the fleet\s*[:\-]?\s*)", re.I)


def fleet_settings(cfg):
    # This node's fleet posture, read from the federation slice
    # of the config — node-local, like lend posture: membership
    # is a property of THIS machine, not an account fact, so it
    # never rides Sync. Defaults: out of the fleet; fan-out cap
    # 4 (the ruled default, operator-settable in config);
    # a 120-second per-ask deadline; composing at home unless
    # the profile marks this node a non-composer with a home
    # node named (Doctrine V).
    raw = {}
    try:
        fed = (cfg or {}).get("federation")
        if isinstance(fed, dict) and isinstance(fed.get("fleet"), dict):
            raw = fed["fleet"]
    except Exception:
        raw = {}

    def _int(key, default, lo, hi):
        try:
            v = int(raw.get(key))
        except Exception:
            return default
        return max(lo, min(hi, v))

    return {
        "member": raw.get("member") is True,
        "cap": _int("cap", 4, 1, 16),
        "deadlineSec": _int("deadlineSec", 120, 1, 600),
        "canCompose": raw.get("canCompose") is not False,
        "composeHome": str(raw.get("composeHome") or "").strip(),
    }


def fleet_public(cfg):
    # The Federation page's view of the posture — the toggle
    # renders from exactly this, and the act answers with it,
    # so flip, stored state, and reload read one truth.
    s = fleet_settings(cfg)
    return {"member": s["member"], "cap": s["cap"],
            "deadlineSec": s["deadlineSec"],
            "canCompose": s["canCompose"],
            "composeHomeSet": bool(s["composeHome"])}


def fleet_set_member(on, cfg):
    # The toggle's stored half. Writes the fleet slice through
    # the raw federation config (preserving the fedinets and
    # the Badge fingerprint, the way the badge acts do), and
    # bumps the member stamp whenever the value actually
    # changes — the stamp is how a lender notices, mid-
    # contribution, that this node has left the fleet.
    rawcfg = dict(_federation_slice(cfg))
    was = fleet_settings(cfg)["member"]
    raw = rawcfg.get("fleet")
    raw = dict(raw) if isinstance(raw, dict) else {}
    raw["member"] = bool(on)
    rawcfg["fleet"] = raw
    cfg["federation"] = rawcfg
    save_config(cfg)
    if bool(on) != was:
        FLEET_STATE["member_stamp"] += 1
    return fleet_settings(cfg)


def fleet_own_name(cfg):
    ms = mesh_settings(cfg)
    return str(ms.get("nodeName") or "").strip() or "This node"


def fleet_run_class(cls, prompt, cfg):
    # Execute ONE whole sub-task of the given class on THIS
    # machine. report and collect are deterministic — the
    # machine's own facts, no model call, so they answer even
    # with a cold model; generate and lookup ride the local
    # model under the class's framing.
    prompt = str(prompt or "")
    if cls == "report":
        ms = mesh_settings(cfg)
        rows = mesh_ledger_read()
        lent = sum(1 for r in rows if r.get("direction") == "lent")
        borrowed = sum(1 for r in rows if r.get("direction") == "borrowed")
        machines = len({str(r.get("peer") or "") for r in rows} - {""})
        lines = [fleet_own_name(cfg) + " reporting in.",
                 "Mesh " + ("on" if ms["enabled"] else "off")
                 + "; lending " + ("on" if ms["lend"]["enabled"] else "off")
                 + "; borrowing "
                 + ("on" if ms["borrow"]["enabled"] else "off")
                 + "; in the fleet: "
                 + ("yes" if fleet_settings(cfg)["member"] else "no") + ".",
                 "Ledger: " + str(lent) + " generations lent, "
                 + str(borrowed) + " borrowed, across "
                 + str(machines) + " machines."]
        return "\n".join(lines)
    if cls == "collect":
        try:
            digest = INTEL.build_digest()
        except Exception:
            digest = None
        if not digest:
            return (fleet_own_name(cfg) + " holds no Wire digest right "
                    "now — the collector has not built one on this "
                    "machine.")
        lines = ["The Wire digest this machine holds (built "
                 + str(digest.get("generated_at") or "at an unrecorded time")
                 + "):"]
        for src in digest.get("sources") or []:
            age = src.get("ageHours")
            lines.append("- " + str(src.get("label") or src.get("source")
                                    or "source")
                         + ": " + str(src.get("count") or 0) + " items"
                         + ((", last gathered " + str(age) + "h ago")
                            if isinstance(age, (int, float)) else ""))
        sections = digest.get("sections") or {}
        for key in ("kev_new", "stack_matches", "top_new"):
            rows = sections.get(key) or []
            if rows:
                lines.append("- " + key.replace("_", " ") + ": "
                             + str(len(rows)) + " entries")
        return "\n".join(lines)
    framing = _FLEET_FRAMING.get(cls) or _FLEET_FRAMING["generate"]
    messages = [{"role": "system", "content": framing},
                {"role": "user", "content": prompt}]
    endpoint = cfg.get("live", {}).get("llamafile_endpoint",
                                       "http://localhost:8080/v1")
    model = cfg.get("live", {}).get("model_name") or "navi"
    reply, _usage = llamafile_chat(messages, endpoint, model)
    return str(reply or "")


def fleet_discover(cfg):
    # The candidate answerers: this node first, then every
    # borrow peer — pairwise, then each joined fedinet's roster
    # doors — each probed once for its fleet membership over
    # the existing /mesh/status door. A peer the probe cannot
    # reach is "unknown": named in a clause it is still asked
    # (the ask itself names the gap); unnamed, it stays out of
    # the rotation.
    own_id = mesh_node_id()
    out = [{"nodeId": own_id, "name": fleet_own_name(cfg),
            "self": True, "member": True, "peer": None}]
    seen = {own_id}
    # The borrow peer list is the whole roster of roads:
    # hand-made pairings and federation members alike (the
    # federation's adopted members stand in it as records,
    # under their roster terms).
    try:
        peers = list(mesh_settings(cfg)["borrow"]["peers"])
    except Exception:
        peers = []
    for peer in peers:
        nid = str(peer.get("nodeId") or "")
        if not nid or nid in seen:
            continue
        seen.add(nid)
        status, _err = mesh_probe_peer(peer)
        member = None
        if isinstance(status, dict):
            member = bool((status.get("fleet") or {}).get("member"))
        out.append({"nodeId": nid,
                    "name": str(peer.get("name") or "A peer"),
                    "self": False, "member": member, "peer": peer})
    return out


def fleet_split_clauses(text):
    # The decomposition's knife: sentence ends, semicolons,
    # and "and also" — the joints an operator actually writes
    # a multi-part ask at. Deliberately conservative: text it
    # cannot split stays one clause, and one clause on the
    # judged route is not a fleet ask at all.
    text = re.sub(r"\s+", " ", str(text or "")).strip()
    if not text:
        return []
    parts = re.split(r"(?<=[.?!])\s+|\s*;\s*|\s+and also\s+", text)
    out = []
    for p in parts:
        p = p.strip().strip(".").strip()
        if len(p.split()) >= 2:
            out.append(p)
    return out


def fleet_classify(clause):
    low = str(clause or "").lower()
    if re.search(r"\bpeek\b|on (your|its|their) screen|screen right now",
                 low):
        return "peek"
    if re.search(r"\b(collect|gather|digest)\b|on the wire|headlines|"
                 r"threat (intel|news)", low):
        return "collect"
    if re.search(r"\b(report|status|health|vitals)\b|"
                 r"how is .{0,40} (doing|running)", low):
        return "report"
    if re.search(r"^(what|what's|who|when|where|which|how|is|are|does|"
                 r"do|did|can|could)\b|look up|find out", low):
        return "lookup"
    return "generate"


def _fleet_named(clause, cand):
    low = str(clause or "").lower()
    name = str(cand.get("name") or "").lower()
    if name and name in low:
        return True
    nid = str(cand.get("nodeId") or "")
    return bool(nid) and nid[:8].lower() in low


def fleet_plan(ask_text, cfg, candidates, explicit=False):
    # Decomposition — deterministic, and a record: who was
    # asked what, under which class, and what was held back or
    # never asked, by name. Returns None when the ask is not a
    # fleet ask at all (a plain single-part question on the
    # judged route — #289: the fleet never seizes it).
    settings = fleet_settings(cfg)
    clauses = fleet_split_clauses(ask_text)
    if not clauses:
        return None
    if not explicit and len(clauses) < 2:
        return None
    rotation = [c for c in candidates if c["member"] is True]
    if not rotation:
        rotation = [candidates[0]]
    subtasks, gaps, held = [], [], []
    ri = 0
    for clause in clauses:
        cls = fleet_classify(clause)
        if cls == "peek":
            held.append({"task": clause, "class": "peek",
                         "sentence": "Held back: \u201c" + clause
                         + "\u201d names a peek \u2014 peeks never fan "
                         "out, under any name."})
            continue
        named = None
        for c in candidates:
            if not c["self"] and _fleet_named(clause, c):
                named = c
                break
        if named is not None and named["member"] is False:
            gaps.append({"name": named["name"], "task": clause,
                         "class": cls,
                         "sentence": named["name"] + " is not in the "
                         "fleet \u2014 its toggle is off, so this piece "
                         "was never asked."})
            continue
        if named is not None:
            target = named
        else:
            target = rotation[ri % len(rotation)]
            ri += 1
        subtasks.append({"id": "s" + str(len(subtasks) + 1),
                         "task": clause, "class": cls,
                         "answererId": target["nodeId"],
                         "answererName": target["name"],
                         "self": bool(target["self"]),
                         "peer": target["peer"]})
    # The fan-out cap counts distinct answerers. Answerers
    # beyond it are not asked at all — the gap is named in the
    # plan, and the plan surfaces (it is no longer silent).
    cap = settings["cap"]
    distinct, kept, overflow = [], [], False
    for st in subtasks:
        if st["answererId"] not in distinct:
            if len(distinct) >= cap:
                overflow = True
                gaps.append({"name": st["answererName"],
                             "task": st["task"], "class": st["class"],
                             "sentence": st["answererName"] + " was not "
                             "asked \u2014 the fan-out cap is " + str(cap)
                             + " machines, and this ask already spans "
                             "it."})
                continue
            distinct.append(st["answererId"])
        kept.append(st)
    surfaced = bool(overflow or held or len(kept) > cap)
    return {"subtasks": kept, "gaps": gaps, "held": held,
            "surfaced": surfaced, "cap": cap}


def fleet_gap_sentence(name, outcome, detail):
    # The named-gap rule: every missing piece is badged with
    # its machine's name and the reason in plain words.
    if outcome == "refusal":
        reason = str(detail or "")
        table = {
            "cap": name + "'s borrow cap for the hour is spent \u2014 "
                   "it could not take this piece.",
            "fleet-off": name + " is not in the fleet \u2014 its "
                         "toggle is off.",
            "fleet-left": name + " left the fleet mid-ask \u2014 this "
                          "piece is missing.",
            "fleet-peek": "A peek never fans out \u2014 " + name
                          + " was never going to be asked this.",
            "fleet-hop": name + " refused a fleet ask inside a fleet "
                         "ask \u2014 one hop is the whole road.",
            "preempted": name + "'s operator needed the machine back "
                         "mid-answer \u2014 preempted.",
            "no-consent": name + " has not ticked lend consent for "
                          "this road.",
            "mesh-off": name + " is not lending right now \u2014 its "
                        "Mesh is off.",
            "lend-off": name + " is not lending right now.",
            "unauthorized": name + " did not accept this node's "
                            "signature.",
            "too-large": "The piece was too large for " + name
                         + "'s door.",
        }
        if reason in table:
            return table[reason]
        return name + " refused this piece (" + (reason or "unnamed") + ")."
    if outcome == "dropped":
        return "The connection to " + name + " dropped mid-answer."
    if outcome == "timeout":
        return name + " did not answer before the fleet deadline."
    if outcome == "error":
        kind = str(detail or "")
        if kind == "timeout":
            return name + " did not answer before the fleet deadline."
        if kind in ("unreachable", "relay-dark", "dest-dark"):
            return name + " is dark \u2014 no answer came back."
        return name + " could not be reached (" + (kind or "error") + ")."
    return name + " did not answer."


def fleet_send_contribution(peer, subtask, ask_id, cfg, timeout):
    # One contribution over the existing roads: a signed POST
    # to the peer's /mesh/borrow door carrying a fleet block
    # (the ask's id, the sub-task's class, depth 0), falling
    # back to a relay when the direct road fails — the same
    # roads, the same consent, as an ordinary borrow. Outcomes
    # mirror the borrower's: ("ok", text) / ("refusal", reason)
    # / ("dropped", None) / ("error", kind).
    messages = [{"role": "system",
                 "content": _FLEET_FRAMING.get(subtask["class"])
                 or _FLEET_FRAMING["generate"]},
                {"role": "user", "content": subtask["task"]}]
    fleet_block = {"ask": ask_id, "class": subtask["class"], "depth": 0}
    request_id = secrets.token_hex(8)
    body = json.dumps({"requestId": request_id, "messages": messages,
                       "model": "navi", "maxTokens": CHAT_MAX_TOKENS,
                       "fleet": fleet_block}).encode()
    kind = "unreachable"
    try:
        code, raw = _mesh_request(peer, "POST", "/mesh/borrow", body,
                                  request_id, timeout)
    except _MeshFpError:
        return ("error", "fingerprint")
    except _MeshTlsError:
        return ("error", "tls")
    except _MeshTimeout:
        kind = "timeout"
    except _MeshDropped:
        return ("dropped", None)
    except _MeshUnreachable:
        kind = "unreachable"
    except Exception:
        kind = "unreachable"
    else:
        if code in (401, 403):
            return ("refusal", "unauthorized")
        try:
            data = json.loads(raw.decode())
        except Exception:
            return ("dropped", None)
        if not isinstance(data, dict):
            return ("dropped", None)
        if data.get("ok"):
            return ("ok", str(data.get("reply") or ""))
        return ("refusal", str(data.get("reason") or "lender-error"))
    # The direct road failed: the relay is the other existing
    # road — one hop, under the relay's own consent rules.
    try:
        relays = mesh_relay_candidates(peer, cfg)
    except Exception:
        relays = []
    for relay in relays:
        out = _mesh_borrow_via_relay(relay, peer, messages,
                                     fleet=fleet_block)
        if out[0] == "ok":
            return ("ok", out[1])
        if out[0] == "refusal":
            return ("refusal", out[1])
        if out[0] == "dropped":
            return ("dropped", None)
        if out[0] == "stage":
            stage = out[1]
            if stage == "dest-dark":
                return ("error", "unreachable")
            if stage == "dest-refused":
                return ("refusal",
                        str((out[2] or {}).get("detail") or "lender-error"))
            if stage == "no-consent":
                return ("refusal", "no-consent")
            return ("error", stage)
    return ("error", kind)


def fleet_record_text(ask_text, plan, sections, gaps):
    # The plan as a standing record: who was asked what, under
    # which class; the contributions with their machine
    # provenance on their face ("According to <machine>
    # (<class>):" — the lead shape, deliberately, so the
    # filing suggestion under this record reads lead); the
    # gaps named. In a case chat the bridge appends this
    # record to the thread, so the standing proposal under it
    # files the plan and its contributions as leads with
    # provenance intact (#291).
    lines = ["Fleet plan \u2014 " + str(ask_text or "")]
    for st in plan["subtasks"]:
        lines.append("\u2022 " + st["task"] + " \u2014 " + st["class"]
                     + " \u2192 " + st["answererName"])
    if sections:
        lines.append("")
        lines.append("Contributions")
        for s in sections:
            lines.append("According to " + s["name"] + " (" + s["class"]
                         + "): " + s["text"])
    if gaps:
        lines.append("")
        lines.append("Named gaps")
        for g in gaps:
            lines.append("\u2014 " + g["sentence"])
    return "\n".join(lines)


def fleet_compose(ask_text, plan, results, cfg, ask_id):
    # Composition — the asker's Navi assembles the answer with
    # the seams visible. The assembly is structural on
    # purpose: each contribution stands under its machine's
    # name and class, and the gaps stand as a named block —
    # never a model rewrite that could smooth a seam (or a
    # missing piece) away. Doctrine V: a node whose profile
    # says it cannot compose sends the assembly job — one
    # more whole generate sub-task, under this ask's id — to
    # its home node; if home cannot answer either, the
    # structural assembly ships with that gap named too.
    settings = fleet_settings(cfg)
    sections, gaps = [], list(plan["gaps"])
    for g in plan["held"]:
        gaps.append({"name": "", "task": g["task"], "class": "peek",
                     "sentence": g["sentence"]})
    for st in plan["subtasks"]:
        out = results.get(st["id"])
        if out is None:
            gaps.append({"name": st["answererName"], "task": st["task"],
                         "class": st["class"],
                         "sentence": st["answererName"] + " did not "
                         "answer before the fleet deadline \u2014 the "
                         "answer ships without that piece."})
        elif out[0] == "ok":
            sections.append({"name": st["answererName"],
                             "class": st["class"], "task": st["task"],
                             "text": str(out[1] or "")})
        else:
            gaps.append({"name": st["answererName"], "task": st["task"],
                         "class": st["class"],
                         "sentence": fleet_gap_sentence(
                             st["answererName"], out[0], out[1])})
    if not settings["canCompose"] and settings["composeHome"]:
        home_peer = None
        try:
            for peer in mesh_settings(cfg)["borrow"]["peers"]:
                if str(peer.get("nodeId") or "") == settings["composeHome"]:
                    home_peer = peer
                    break
        except Exception:
            home_peer = None
        if home_peer is not None:
            home_name = str(home_peer.get("name") or "Home")
            prompt = ("Compose one answer to the operator's ask from "
                      "these signed fleet contributions. Keep each "
                      "machine's contribution attributable to its "
                      "machine; do not invent missing pieces; name "
                      "the gaps as gaps.\n\nThe ask: " + str(ask_text)
                      + "\n\n" + fleet_record_text(ask_text, plan,
                                                   sections, gaps))
            out = fleet_send_contribution(
                home_peer, {"class": "generate", "task": prompt},
                ask_id, cfg, settings["deadlineSec"])
            if out[0] == "ok" and str(out[1] or "").strip():
                return {"answer": str(out[1]), "composedBy": home_name,
                        "sections": sections, "gaps": gaps}
            gaps.append({"name": home_name, "task": "compose this answer",
                         "class": "generate",
                         "sentence": home_name + " could not compose "
                         "this answer \u2014 it is assembled here "
                         "instead, seams as they stand."})
    lines = []
    for s in sections:
        lines.append("**" + s["name"] + "** \u00b7 " + s["class"])
        lines.append(s["text"])
        lines.append("")
    if gaps:
        lines.append("**Named gaps**")
        for g in gaps:
            lines.append("\u2014 " + g["sentence"])
    answer = "\n".join(lines).strip()
    if not answer:
        answer = ("The fleet answered with nothing \u2014 every piece "
                  "is a named gap above.")
    return {"answer": answer, "composedBy": fleet_own_name(cfg),
            "sections": sections, "gaps": gaps}


def fleet_ask(ask_text, cfg, explicit=False, chat_id=None):
    # Run one fleet ask end to end: plan, fan out in parallel
    # (the wall clock becomes the longest contribution, not
    # the sum — the speed case for the weakest node), compose
    # with the seams showing. Returns None when the ask is not
    # a fleet ask; a refused dict when the machinery itself
    # says no; else the run record.
    if FLEET_STATE["executing"] > 0:
        return {"refused": "A fleet ask never fans out from inside a "
                "fleet contribution \u2014 one hop, hard."}
    settings = fleet_settings(cfg)
    candidates = fleet_discover(cfg)
    plan = fleet_plan(ask_text, cfg, candidates, explicit=explicit)
    if plan is None:
        return None
    ask_id = "fleet-" + secrets.token_hex(6)
    deadline = time.time() + settings["deadlineSec"]
    results = {}
    lock = threading.Lock()

    def run_one(st):
        started = time.time()
        if st["self"]:
            try:
                out = ("ok", fleet_run_class(st["class"], st["task"], cfg))
            except Exception:
                out = ("error", "local")
        else:
            remaining = deadline - time.time()
            if remaining <= 0:
                out = ("timeout", None)
            else:
                out = fleet_send_contribution(
                    st["peer"], st, ask_id, cfg,
                    max(1, int(min(remaining, CHAT_TIMEOUT))))
            if out[0] == "ok":
                try:
                    mesh_ledger_append({
                        "peer": st["answererId"], "direction": "borrowed",
                        "wallMs": int((time.time() - started) * 1000),
                        "outcome": "ok",
                        "fleetAsk": ask_id, "class": st["class"]})
                except Exception:
                    pass
        with lock:
            results[st["id"]] = out

    FLEET_STATE["executing"] += 1
    try:
        threads = []
        for st in plan["subtasks"]:
            t = threading.Thread(target=run_one, args=(st,), daemon=True)
            t.start()
            threads.append(t)
        for t in threads:
            left = deadline - time.time()
            if left <= 0:
                break
            t.join(left)
    finally:
        FLEET_STATE["executing"] -= 1
    composed = fleet_compose(ask_text, plan, results, cfg, ask_id)
    return {"ask": ask_id, "plan": plan, "results": results,
            "answer": composed["answer"],
            "composedBy": composed["composedBy"],
            "gaps": composed["gaps"], "sections": composed["sections"]}


def fleet_chat_reply(last, cfg, chat_id=None):
    # The chat door. Two ways in: the explicit trigger
    # ("fleet: ..." / "ask the fleet ...") — the operator
    # invoking the machinery by name — and Navi's judgment on
    # a plainly multi-part ask. A plain single-part question
    # NEVER routes here (#289's lock, kept): it falls through
    # to the ordinary answer untouched. An explicit trigger on
    # a node that is not in the fleet gets the honest named
    # answer, not a silent fall-through — the operator asked
    # for machinery, and the bridge says why it cannot run.
    text = str(last or "").strip()
    if not text:
        return None
    m = _FLEET_TRIGGER.match(text)
    explicit = bool(m)
    ask = text[m.end():].strip() if explicit else text
    if not ask:
        return None
    settings = fleet_settings(cfg)
    if not settings["member"]:
        if explicit:
            return {"reply": "I can't run a fleet ask \u2014 this node "
                    "is not in the fleet. Fleet Answers is a toggle on "
                    "the Federation page, off by default; joining is "
                    "this node's own act, and the fleet only ever "
                    "spans machines already paired or on a joined "
                    "fedinet's roster."}
        return None
    if not explicit and len(fleet_split_clauses(ask)) < 2:
        return None
    run = fleet_ask(ask, cfg, explicit=explicit, chat_id=chat_id)
    if run is None:
        return None
    if run.get("refused"):
        return {"reply": run["refused"]}
    plan = run["plan"]
    reply = run["answer"]
    if plan["surfaced"]:
        head = ["The fleet plan for that ask:"]
        for st in plan["subtasks"]:
            head.append("\u2022 " + st["task"] + " \u2014 " + st["class"]
                        + " \u2192 " + st["answererName"])
        reply = "\n".join(head) + "\n\n" + reply
    # Case tie-in (#286/#291): in a case chat the plan and its
    # contributions stand in the thread as their own record —
    # appended by the bridge, like every receipt — so the
    # standing proposal under it files them as LEADS with
    # machine provenance; the composed answer above is this
    # turn's reply, finding-shaped. Nothing files itself.
    if chat_id is not None:
        try:
            chat = None
            for rec in chats_all():
                if str(rec.get("id")) == str(chat_id):
                    chat = rec
                    break
            if chat is not None and chat.get("case"):
                record = fleet_record_text(ask, plan, run["sections"],
                                           run["gaps"])
                now_ms = int(time.time() * 1000)
                msgs = list(chat.get("messages") or [])
                msgs.append({"who": "NAVI", "text": record, "exec": None,
                             "ts": now_ms, "fleetAsk": run["ask"]})
                chat["messages"] = msgs
                chat["updatedAt"] = now_ms
                chat_write(chat)
        except Exception:
            pass
    return {"reply": reply,
            "fleet": {"ask": run["ask"],
                      "contributions": len(run["sections"]),
                      "gaps": len(run["gaps"]),
                      "composedBy": run["composedBy"]}}


# -- Borrower transport ----------------------------------------------
# TLS with verification OFF and the peer card's fingerprint PINNED:
# the borrower compares the presented certificate's DER hash
# against the card by hand and refuses a mismatch in plain words.
# The pin is the authentication; no CA is involved (spec 4.1).

class _MeshFpError(Exception):
    pass


class _MeshDropped(Exception):
    pass


class _MeshUnreachable(Exception):
    pass


class _MeshTlsError(Exception):
    # 0.2.0e Build 1 (#143): an address answered, but the TLS
    # handshake itself failed — whatever is listening there is
    # not speaking the Mesh door's TLS. Its own kind so a failure
    # can name its stage instead of collapsing into "unreachable".
    pass


class _MeshTimeout(Exception):
    # 0.2.0e Build 1 (#143): no address answered inside the
    # timeout — its own kind, for the same reason as above.
    pass


# Build .87 (gate 4, borrower half): the generous cap on a borrow
# read, in seconds. Connection TERMINATION is the drop signal — a
# lender that stops mid-generation hard-closes its socket and this
# borrower's blocked read fails at once (see _mesh_request). This
# cap is only the backstop for a lender that vanishes WITHOUT
# closing (a dead network path, a frozen process): far beyond a
# healthy generation's normal span, far short of the 600s chat
# timeout, and its expiry is classified exactly like termination —
# dropped, so the fusion fallback runs.
MESH_BORROW_READ_CAP = 330


def _mesh_request(peer, method, path, body, request_id, timeout):
    # One signed Mesh request: this node signs, _mesh_dial dials.
    return _mesh_dial(peer, method, path, body, timeout,
                      mesh_signed_headers(request_id, body or b""))


def _mesh_send_prepared(peer, method, path, headers, body, timeout):
    # One Mesh request carrying headers the CALLER prepared —
    # 0.2.0e Build 7 (#154): the relay's onward leg. The inner
    # request a relay forwards is the asker's own, byte-identical
    # including its signature headers; the relay signs nothing of
    # its own onto it. Same dial semantics as _mesh_request
    # (addresses in card order, fingerprint pinned from THIS
    # machine's record for the destination, same error kinds).
    return _mesh_dial(peer, method, path, body, timeout, headers)


def _mesh_dial(peer, method, path, body, timeout, headers):
    # The shared dial: try the peer's addresses in card order,
    # presenting `headers` verbatim. Returns (status, raw-body).
    # Raises _MeshFpError when
    # an address answered but none presented the pinned
    # fingerprint, _MeshDropped when a connected request died
    # mid-flight, _MeshTlsError when an address answered but the
    # TLS handshake failed, _MeshTimeout when nothing answered in
    # time, _MeshUnreachable when no address could be connected
    # at all. (0.2.0e Build 1, #143: the failure kinds name their
    # stage — the classification is reporting only; which error
    # a caller acts on, and how, is unchanged.)
    last_err = None
    fp_err = None
    tls_err = None
    timeout_err = None
    # Build 6 (#226): the dial remembers WHICH address each
    # failure kind died at, and the classified exceptions carry
    # it (mesh_kind / mesh_addr) — the federation sentences name
    # the stage and the road instead of flattening every failure
    # into "did not answer".
    fp_addr = ""
    tls_addr = ""
    timeout_addr = ""
    last_addr = ""
    for address in _mesh_effective_addresses(peer):
        # Build 4 (#143/#154): an entry may carry its own port
        # (host:port); a bare host inherits the record's port.
        # One helper parses both, here and in the health sweep.
        host, eport = _mesh_addr_parts(address, peer["port"])
        disp = host + ":" + str(eport)
        last_addr = disp
        conn = None
        sent = False
        try:
            ctx = ssl._create_unverified_context()
            conn = http.client.HTTPSConnection(host, eport, timeout=timeout, context=ctx)
            conn.connect()
            der = b""
            try:
                der = conn.sock.getpeercert(binary_form=True) or b""
            except Exception:
                der = b""
            fp = hashlib.sha256(der).hexdigest() if der else ""
            if peer["certFingerprint"] and fp != peer["certFingerprint"]:
                conn.close()
                # Build 7 (#87): a mismatch at ONE address no
                # longer aborts the whole request. A card can
                # carry several addresses and something else may
                # be answering at the wrong one — record the
                # mismatch and try the next address. _MeshFpError
                # is raised only at the end, when no address both
                # connected AND matched (see below).
                fp_err = "fingerprint mismatch"
                fp_addr = disp
                last_err = _MeshFpError("fingerprint mismatch")
                continue
            conn.request(method, path, body=body, headers=headers)
            sent = True
            if path in ("/mesh/borrow", "/mesh/relay"):
                # Build .87 (gate 4, borrower half): bound the
                # borrow read by MESH_BORROW_READ_CAP (see its
                # comment). When the lender terminates the
                # connection mid-borrow — the drop path's socket
                # shutdown — getresponse()/read() below raises at
                # once (RemoteDisconnected / reset / EOF), and the
                # sent-branch of the except maps ANY such failure,
                # cap expiry included, to _MeshDropped. No short
                # timeout is imposed: a healthy lender generation
                # may legitimately take minutes.
                try:
                    if conn.sock is not None:
                        conn.sock.settimeout(min(timeout, MESH_BORROW_READ_CAP))
                except Exception:
                    pass
            resp = conn.getresponse()
            raw = resp.read()
            status = resp.status
            conn.close()
            return status, raw
        except _MeshFpError:
            raise
        except Exception as e:
            try:
                if conn is not None:
                    conn.close()
            except Exception:
                pass
            if sent:
                dropped = _MeshDropped(str(e))
                dropped.mesh_kind = "dropped"
                dropped.mesh_addr = disp
                raise dropped
            if isinstance(e, ssl.SSLError):
                tls_err = str(e)
                tls_addr = disp
            elif isinstance(e, (socket.timeout, TimeoutError)):
                timeout_err = str(e)
                timeout_addr = disp
            last_err = e
            continue
    # Build 7 (#87): something DID answer at one of the addresses
    # but never with the pinned certificate — that is the honest,
    # diagnosable failure, and it outranks a plain unreachable.
    if fp_err is not None:
        err = _MeshFpError(fp_err)
        err.mesh_kind = "fingerprint"
        err.mesh_addr = fp_addr
        raise err
    if tls_err is not None:
        err = _MeshTlsError(tls_err)
        err.mesh_kind = "tls"
        err.mesh_addr = tls_addr
        raise err
    if timeout_err is not None:
        err = _MeshTimeout(timeout_err)
        err.mesh_kind = "timeout"
        err.mesh_addr = timeout_addr
        raise err
    err = _MeshUnreachable(str(last_err))
    err.mesh_kind = "unreachable"
    err.mesh_addr = last_addr
    raise err


def _mesh_failure_stage(exc):
    # Build 6 (#226): the failure as a sentence clause naming the
    # stage and the road it died on — "the TLS handshake failed
    # at host:port" — or "" when the exception carries no dial
    # classification (not a transport failure at all). The
    # federation knock and roster-sync sentences speak this
    # clause; before this build they flattened every transport
    # failure into "did not answer", and a joiner's audit log
    # showed nothing at all for a failed knock, which read as
    # "no knock was ever sent" during a live diagnosis.
    kind = getattr(exc, "mesh_kind", None)
    if kind is None:
        for cls, k in ((_MeshFpError, "fingerprint"),
                       (_MeshTlsError, "tls"),
                       (_MeshTimeout, "timeout"),
                       (_MeshDropped, "dropped"),
                       (_MeshUnreachable, "unreachable")):
            if isinstance(exc, cls):
                kind = k
                break
    addr = getattr(exc, "mesh_addr", "") or ""
    if kind == "timeout":
        return ("the last road tried, " + addr + ", did not "
                "answer in time") if addr else "nothing answered in time"
    if kind == "unreachable":
        return ("the last road tried, " + addr + ", could not "
                "be connected to") if addr else "no address could be connected to"
    if kind == "tls":
        return ("the TLS handshake failed at " + addr) if addr \
            else "the TLS handshake failed"
    if kind == "fingerprint":
        return ("the door that answered at " + addr + " did not "
                "present the pinned certificate") if addr \
            else "the door that answered did not present the pinned certificate"
    if kind == "dropped":
        return ("the connection to " + addr + " dropped "
                "mid-answer") if addr else "the connection dropped mid-answer"
    return ""


def mesh_probe_peer(peer, force=False):
    # GET /mesh/status with a short timeout, cached briefly.
    # Returns (status-dict-or-None, error-kind).
    cache = MESH_STATE["probe_cache"].get(peer["nodeId"])
    now = time.time()
    if not force and cache and now - cache[0] < 20:
        return cache[1], cache[2]
    status = None
    err = "unreachable"
    try:
        code, raw = _mesh_request(peer, "GET", "/mesh/status", b"", "", 3)
        if code == 200:
            try:
                parsed = json.loads(raw.decode())
                if isinstance(parsed, dict) and parsed.get("ok"):
                    status = parsed
            except Exception:
                status = None
        elif code in (401, 403):
            err = "unauthorized"
    except _MeshFpError:
        err = "fingerprint"
    except _MeshTlsError:
        err = "tls"
    except _MeshTimeout:
        err = "timeout"
    except Exception:
        err = "unreachable"
    # Build 7 gate repair: the cache carries a fourth element —
    # the stamp the roster's "(last checked … ago)" age is
    # measured from. A successful probe stamps it with now; a
    # FAILED probe keeps the previous stamp, updating only the
    # error kind, so during a long outage the age keeps growing
    # (how long the peer has been dark) instead of cycling with
    # the ~30s health cadence. A peer that has never been
    # reached has no success to keep: its first failed probe
    # stamps it, and the age counts from that first check. The
    # first element stays the latest ATTEMPT's time — the 20s
    # borrow-cache freshness reads exactly what it read before.
    if status is not None:
        ok_stamp = now
    elif cache is not None and len(cache) > 3 and cache[3]:
        ok_stamp = cache[3]
    else:
        ok_stamp = now
    MESH_STATE["probe_cache"][peer["nodeId"]] = (now, status, "" if status is not None else err, ok_stamp)
    return status, "" if status is not None else err


# -- Background peer health (Build 7, #87) ---------------------------
# Before this build the probe above had exactly one caller — a
# chat borrow — so a paired peer's state existed only as a side
# effect of borrowing, and a pure lender (or a pair nobody had
# chatted across yet) sat at "not checked yet" forever, which the
# first field pair did. The bridge now watches its peers itself:
# a daemon thread probes each paired peer on a ~30s cadence while
# Mesh is enabled, and pairing / Mesh enable probe at once. Only
# state TRANSITIONS earn a log line, with the error kind — the
# log records that a peer went dark or came back, never a
# drumbeat of successful checks. Peer names only in those lines,
# never an address, like every Mesh surface.

def mesh_probe_tracked(peer):
    # One forced probe whose result is compared with the last
    # health state the bridge logged for the peer; a change is
    # written to the bridge log, anything else stays silent.
    status, err = mesh_probe_peer(peer, force=True)
    cur = "reachable" if status is not None else "unreachable:" + (err or "unreachable")
    with MESH_LOCK:
        prev = MESH_STATE["peer_health"].get(peer["nodeId"])
        if prev != cur:
            MESH_STATE["peer_health"][peer["nodeId"]] = cur
    if prev == cur:
        return status, err
    name = peer["name"]
    if status is not None:
        if prev is None:
            print("Navi Mesh: peer " + name + " checked \u2014 reachable.", flush=True)
        else:
            print("Navi Mesh: peer " + name + " is reachable again.", flush=True)
    else:
        kind = err or "unreachable"
        if prev is None:
            print("Navi Mesh: peer " + name + " checked \u2014 unreachable (" + kind + ").", flush=True)
        elif prev == "reachable":
            print("Navi Mesh: peer " + name + " is now unreachable (" + kind + ").", flush=True)
        else:
            print("Navi Mesh: peer " + name + " is still unreachable \u2014 the reason is now " + kind + ".", flush=True)
    return status, err


def mesh_probe_all_tracked():
    # Probe every paired peer once (tracked). A no-op while Mesh
    # is off — pairing can be recorded with the master switch in
    # any state, and a disabled Mesh probes nothing.
    cfg = load_config()
    ms = mesh_settings(cfg)
    if not ms["enabled"]:
        return
    for peer in ms["borrow"]["peers"]:
        try:
            mesh_probe_tracked(peer)
        except Exception:
            pass


def mesh_probe_peers_soon():
    # An immediate background probe of every paired peer: pairing
    # and Mesh enable call this so a fresh peer is checked at
    # once instead of waiting for the health cadence. Never
    # blocks the caller — the probes run on their own thread.
    t = threading.Thread(target=mesh_probe_all_tracked, daemon=True)
    t.start()


def mesh_health_loop():
    # The cadence itself. The loop runs for the bridge's life
    # once started and simply idles — no probes, no log — while
    # Mesh is off or nothing is paired.
    while True:
        try:
            mesh_probe_all_tracked()
        except Exception:
            pass
        time.sleep(30)


def mesh_ensure_health_thread():
    with MESH_LOCK:
        if MESH_STATE["health_started"]:
            return
        MESH_STATE["health_started"] = True
    t = threading.Thread(target=mesh_health_loop, daemon=True)
    t.start()


def _mesh_borrow_once(peer, messages):
    # A single borrow attempt against a peer already probed
    # reachable. Outcomes: ("ok", reply, wallMs), ("refusal",
    # reason), ("dropped", None) for an in-flight failure (R3),
    # ("error", kind) for a failure before admission.
    request_id = secrets.token_hex(8)
    body = json.dumps({
        "requestId": request_id,
        "messages": messages,
        "model": "navi",
        "maxTokens": CHAT_MAX_TOKENS,
    }).encode()
    started = time.time()
    try:
        code, raw = _mesh_request(peer, "POST", "/mesh/borrow", body, request_id, CHAT_TIMEOUT)
    except _MeshFpError:
        return ("error", "fingerprint")
    except _MeshTlsError:
        return ("error", "tls")
    except _MeshTimeout:
        return ("error", "timeout")
    except _MeshDropped:
        # Build .87: remember the drop for the status payload —
        # the Servers row's Mesh warning ("<name> dropped") reads
        # it until a later probe finds the peer healthy again.
        with MESH_LOCK:
            MESH_STATE["peer_dropped"][peer["nodeId"]] = time.time()
        return ("dropped", None)
    except _MeshUnreachable:
        return ("error", "unreachable")
    except Exception:
        return ("error", "unreachable")
    if code in (401, 403):
        return ("refusal", "unauthorized")
    try:
        data = json.loads(raw.decode())
    except Exception:
        with MESH_LOCK:
            MESH_STATE["peer_dropped"][peer["nodeId"]] = time.time()
        return ("dropped", None)
    if code == 200 and isinstance(data, dict) and data.get("ok"):
        wall = data.get("wallMs")
        if not isinstance(wall, (int, float)):
            wall = int((time.time() - started) * 1000)
        return ("ok", str(data.get("reply") or ""), int(wall))
    reason = str((data or {}).get("reason") or "lender-error") if isinstance(data, dict) else "lender-error"
    return ("refusal", reason)


_RELAY_SPEED_RANK = {"GPU-accelerated": 0, "Integrated graphics": 1, "CPU": 2}


def _mesh_relay_candidates(ms, dest):
    # This machine's consented carriers for one destination: the
    # peers (other than the destination itself) whose record here
    # ticks that they may carry this machine's envelopes to the
    # destination. Card order; placement sorts them at ask time.
    out = []
    for p in ms["borrow"]["peers"]:
        if p["nodeId"] == dest["nodeId"]:
            continue
        if dest["nodeId"] in (p.get("relayCarryTo") or []):
            out.append(p)
    return out


def _mesh_borrow_via_relay(relay, dest, messages, fleet=None):
    # One relayed borrow attempt. Builds the inner request
    # exactly as _mesh_borrow_once does — the destination could
    # not tell it from a direct ask but for the signed body's
    # "via", naming the carrier — captures its signed headers,
    # and wraps both in the outer envelope this machine signs to
    # the relay. Outcomes mirror _mesh_borrow_once, plus the
    # relay stages: ("stage", name, detail-dict) for a failure
    # the relay named (no-consent / relay-dark / dest-dark /
    # dest-refused / dest-unknown / hop-limit).
    request_id = secrets.token_hex(8)
    _inner = {
        "requestId": request_id,
        "messages": messages,
        "model": "navi",
        "maxTokens": CHAT_MAX_TOKENS,
        "via": relay["nodeId"],
    }
    if fleet is not None:
        # v0.6.1e (#293): a relayed fleet contribution carries
        # its fleet block inside the sealed inner body, exactly
        # as on the direct road.
        _inner["fleet"] = fleet
    inner_body = json.dumps(_inner).encode()
    inner_headers = mesh_signed_headers(request_id, inner_body)
    outer_id = secrets.token_hex(8)
    outer = json.dumps({
        "requestId": outer_id,
        "to": dest["nodeId"],
        "askerConsent": True,
        "inner": {
            "path": "/mesh/borrow",
            "requestId": request_id,
            "headers": inner_headers,
            "bodyB64": base64.b64encode(inner_body).decode("ascii"),
        },
    }).encode()
    started = time.time()
    try:
        code, raw = _mesh_request(relay, "POST", "/mesh/relay",
                                  outer, outer_id, CHAT_TIMEOUT)
    except _MeshFpError:
        return ("error", "fingerprint")
    except _MeshTlsError:
        return ("error", "tls")
    except _MeshTimeout:
        return ("stage", "relay-dark", {})
    except _MeshDropped:
        return ("stage", "relay-dark", {})
    except _MeshUnreachable:
        return ("stage", "relay-dark", {})
    except Exception:
        return ("stage", "relay-dark", {})
    if code in (401, 403):
        return ("refusal", "unauthorized")
    try:
        data = json.loads(raw.decode())
    except Exception:
        return ("stage", "relay-dark", {})
    if not isinstance(data, dict):
        return ("stage", "relay-dark", {})
    if data.get("ok") and data.get("relayed"):
        # The destination's own answer, verbatim: classify it
        # exactly as a direct answer is classified — its refusal
        # words are its own, and a missing lender tick arrives as
        # the no-consent stage it is.
        try:
            inner_raw = base64.b64decode(str(data.get("bodyB64") or ""),
                                         validate=True)
            ans = json.loads(inner_raw.decode())
        except Exception:
            return ("dropped", None)
        if isinstance(ans, dict) and ans.get("ok"):
            wall = ans.get("wallMs")
            if not isinstance(wall, (int, float)):
                wall = int((time.time() - started) * 1000)
            return ("ok", str(ans.get("reply") or ""), int(wall))
        reason = str((ans or {}).get("reason") or "lender-error") \
            if isinstance(ans, dict) else "lender-error"
        if reason == "no-consent":
            return ("stage", "no-consent", {"missing": "lender"})
        return ("refusal", reason)
    reason = str(data.get("reason") or "")
    if reason == "no-consent":
        return ("stage", "no-consent",
                {"missing": str(data.get("missing") or "relay")})
    if reason in ("dest-dark", "dest-refused", "dest-unknown",
                  "hop-limit", "class-off", "too-large", "mesh-off"):
        return ("stage", reason, {"detail": str(data.get("detail") or "")})
    if reason == "dest-dropped":
        return ("dropped", None)
    return ("stage", "relay-dark", {})


def peer_fedinet_id(cfg, peer, fedinets=None):
    # v0.4.2e (#234): the ONE federation recognition every reader
    # shares — the route pill's availability, the status rail's
    # FediNet line, the answer badges, the borrow's own road
    # choice. A peer belongs to a fedinet by ROSTER MEMBERSHIP,
    # never by the stamp its record happened to be created with:
    # the stored viaFedinet when it names a fedinet this node
    # still belongs to, else the node's place on a joined
    # fedinet's signed roster — an ordinary member entry, or the
    # fedinet's gateway itself (the gateway is a member of its
    # own fedinet and never appears as an ordinary roster entry,
    # which is exactly how a hand-paired gateway record stayed
    # invisible to the federation in the field: neondrip's
    # records all carried "viaFedinet": "" because the pairing
    # predated Rockport Mesh and no adoption ever landed).
    # Returns the fedinet's id, or "" when the peer is this
    # node's own pairwise business only.
    if fedinets is None:
        fedinets = federation_settings(cfg)
    fid = str((peer or {}).get("viaFedinet") or "")
    if fid and any(f["id"] == fid for f in fedinets):
        return fid
    nid = str((peer or {}).get("nodeId") or "")
    if not nid:
        return ""
    for f in fedinets:
        if nid == f["gatewayNodeId"] or _fed_member(f, nid) is not None:
            return f["id"]
    return ""


def _peer_federation_made(peer):
    # A record the federation itself created (roster
    # materialization, a gateway card's introduction): autoPaired
    # is its birthmark. A hand pairing the federation later
    # marked keeps autoPaired as it stood — the federation marked
    # the record, it did not make it — so a dual-hat machine (a
    # hand pairing that is also a roster member) counts for BOTH
    # roads: it is borrowed from as a Mesh peer when the Mesh
    # road is chosen, and recognized as a federation member when
    # the federation road is. Only a federation-MADE record is
    # excluded from the Mesh road, exactly as the stamp
    # partition had it.
    return bool((peer or {}).get("autoPaired"))


def _mesh_route_fields(peer, federation_leg=True):
    # v0.3.0e Build 2 (#203): the federation leg of a served
    # borrow's recorded route — the fedinet under which this
    # borrow was authorized, named from the peer's federation
    # recognition (v0.4.2e, #234: peer_fedinet_id — roster
    # membership, not the record's stamp) resolved against the
    # federation state. A peer with no federation recognition
    # gets no fields: the pill renders the RECORDED route only,
    # and a route with no federation leg is a pairwise route,
    # never dressed up as one. federation_leg=False is the
    # caller's word that the road actually taken was the
    # pairwise Mesh (an explicit Mesh route that a dual-hat
    # machine served): the recorded route is the road taken,
    # not every road the machine could have offered.
    if not federation_leg:
        return {}
    fed_id = peer_fedinet_id(load_config(), peer or {})
    if not fed_id:
        return {}
    try:
        _idx, fed = _fed_find(federation_settings(load_config()), fed_id)
    except Exception:
        return {}
    if fed is None:
        return {}
    return {"fedinet": fed["name"], "fedinetId": fed["id"]}


def _mesh_relay_fallback(dest, messages, ms, federation_leg=True):
    # The relay road for one dark destination. Returns
    # mesh_try_borrow's own return shapes — (reply, payload) on a
    # relayed answer, (None, payload) when the attempt settles it
    # (the destination answered through the relay and refused, or
    # dropped), and None when no consented carrier exists at all
    # (the caller keeps its direct-road behavior, word for word).
    candidates = _mesh_relay_candidates(ms, dest)
    if not candidates:
        return None
    # Placement among carriers (ring-2 design section 9's open
    # question, read capability-first as the prep recommends):
    # a reachable carrier first, then the one not busy, then the
    # faster class. Every candidate costs at most one short
    # probe before it is asked; a dark carrier is one probe, not
    # a timeout spiral.
    probed = []
    for idx, relay in enumerate(candidates):
        status, _err = mesh_probe_peer(relay)
        caps = relay.get("capabilities") or {}
        if status is not None and status.get("capabilities"):
            live = _mesh_caps_sanitize(status.get("capabilities"))
            if live is not None:
                caps = live
        rank = _RELAY_SPEED_RANK.get(str(caps.get("speedClass") or ""), 3)
        probed.append((status is None, bool(status and status.get("busy")),
                       rank, idx, relay))
    probed.sort(key=lambda t: (t[0], t[1], t[2], t[3]))
    stages = []
    for dark, _busy, _rank, _idx, relay in probed:
        if dark:
            stages.append(("relay-dark", relay, {}))
            continue
        outcome = _mesh_borrow_via_relay(relay, dest, messages)
        if outcome[0] == "ok":
            with MESH_LOCK:
                MESH_STATE["peer_dropped"].pop(dest["nodeId"], None)
            mesh_ledger_append({"peer": dest["nodeId"],
                                "direction": "borrowed",
                                "wallMs": outcome[2], "outcome": "ok",
                                "via": relay["nodeId"], "path": "relay"})
            return outcome[1], {"servedBy": "mesh", "node": dest["name"],
                                "fellBack": False,
                                "viaRelay": relay["name"], "path": "relay",
                                **_mesh_route_fields(dest, federation_leg)}
        if outcome[0] == "dropped":
            return None, {
                "servedBy": "local", "fellBack": True,
                "reason": "lender-dropped", "node": dest["name"],
                "viaRelay": relay["name"], "path": "relay",
                "notice": dest["name"] + " dropped mid-answer \u2014 this reply was generated locally instead.",
            }
        if outcome[0] == "refusal":
            # The destination answered through the carrier and its
            # answer stands — in its own refusal words, with the
            # path named beside them.
            return None, {"servedBy": "local", "fellBack": True,
                          "reason": outcome[1], "node": dest["name"],
                          "viaRelay": relay["name"], "path": "relay"}
        if outcome[0] == "error":
            stages.append((outcome[1], relay, {}))
            continue
        stages.append((outcome[1], relay, outcome[2] or {}))
    # Nothing carried. The named stage is the most actionable one
    # the attempts produced: a missing consent an operator can
    # tick outranks a refusal, which outranks a dark destination,
    # which outranks a dark relay.
    for want in ("no-consent", "hop-limit", "dest-refused",
                 "dest-unknown", "dest-dark", "fingerprint", "tls",
                 "class-off", "too-large", "mesh-off", "relay-dark"):
        for stage, relay, detail in stages:
            if stage == want:
                payload = {"servedBy": "local", "fellBack": True,
                           "reason": stage, "node": dest["name"],
                           "viaRelay": relay["name"], "path": "relay"}
                if detail.get("missing"):
                    payload["missing"] = detail["missing"]
                if detail.get("detail"):
                    payload["detail"] = detail["detail"]
                return None, payload
    return None, {"servedBy": "local", "fellBack": True,
                  "reason": "relay-dark", "node": dest["name"],
                  "path": "relay"}


def _mesh_relay_probe(relay, dest_id):
    # The health check's relay question, asked over the wire: is
    # the relay's own carry tick in place for (this asker, that
    # destination), and is the destination paired there? Returns
    # ("consent" | "no-consent" | "dark" | "unauthorized").
    request_id = secrets.token_hex(8)
    body = json.dumps({"requestId": request_id, "probe": True,
                       "to": dest_id}).encode()
    try:
        code, raw = _mesh_request(relay, "POST", "/mesh/relay",
                                  body, request_id, 3)
    except Exception:
        return "dark"
    if code in (401, 403):
        return "unauthorized"
    try:
        data = json.loads(raw.decode())
    except Exception:
        return "dark"
    if not isinstance(data, dict) or not data.get("ok"):
        return "dark"
    if not data.get("destPaired"):
        return "dest-unknown"
    return "consent" if data.get("consent") else "no-consent"


def _mesh_try_borrow_inner(messages, cfg, ms, route=None):
    # The borrower (spec section 2): first reachable peer only —
    # Ring 1 never races lenders. Returns (reply, mesh-payload) on
    # a mesh-served answer, or (None, mesh-payload) describing the
    # local fallback the caller is about to run. (None, None) when
    # Mesh does not apply at all. `route` (v0.4.1e, #228) is the
    # chat's stored route: it narrows the lender field below, once
    # the borrower's own list is sanitized. mesh_try_borrow (the
    # wrapper after this function) is the public door; this is
    # the engine it drives.
    peers = ms["borrow"]["peers"]
    if not peers:
        return None, None
    # (v0.3.0e — Build 6, #224, the mesh pin): the borrower's own
    # list is sanitized BEFORE it rides — a poisoned stored
    # thread must never reach a lender's model, least of all a
    # lender running an older build without the builder guard.
    # An all-hollow list borrows nothing: the local path's
    # builder names the refusal instead.
    try:
        messages = _chat_sanitize_messages(messages)
    except Exception:
        return None, None
    # v0.4.1e (#228): the route narrows the field — a Mesh route
    # borrows from hand-paired machines only; a Federation route
    # from that fedinet's members only. A route whose field is
    # empty answers locally WITH THE ROUTE NAMED on the payload,
    # so the line under the reply blames the route, not a machine
    # that was never asked.
    # v0.4.2e (#234): membership is RECOGNIZED, not stamped —
    # peer_fedinet_id reads the roster, so a hand-paired gateway
    # is a federation road; and the Mesh field is the hand
    # pairings (a federation-MADE record is not one), so a
    # dual-hat machine stands in both fields, as it should.
    # v0.4.2e (#231): with NO stored route the field is ORDERED
    # by the automatic precedence — federation-recognized
    # machines first, in their stored order, then the rest — so
    # the first reachable machine is the precedence's answer.
    _fed_leg = not (route and route.get("mode") == "mesh")
    if route:
        _mode = route.get("mode")
        if _mode == "mesh":
            peers = [p for p in peers if not _peer_federation_made(p)]
        elif _mode == "federation":
            _fid = route.get("fedinetId") or ""
            peers = [p for p in peers if peer_fedinet_id(cfg, p) == _fid]
        if not peers:
            return None, {"servedBy": "local", "fellBack": True,
                          "reason": "route-unavailable",
                          "routeMode": _mode,
                          "routeName": route.get("name") or ""}
    else:
        peers = sorted(peers,
                       key=lambda p: 0 if peer_fedinet_id(cfg, p) else 1)
    if not mesh_node_id():
        ok, _err = mesh_ensure_identity(cfg)
        if not ok:
            return None, None
    with MESH_LOCK:
        MESH_STATE["borrow_out"] = True
    try:
        last_reason = "unreachable"
        relay_target = None
        for peer in peers:
            # Build 7 (#87): canBorrow is enforced HERE, on the
            # borrowing side — this machine does not borrow from
            # a peer whose flag is off. It no longer gates the
            # lender's authentication (see mesh_authenticate):
            # pairing is that side's consent.
            if not peer["canBorrow"]:
                continue
            status, err = mesh_probe_peer(peer)
            if status is None and err == "unreachable":
                # Build 7 gate repair (close-out): the answer above
                # may be the 20s probe cache holding a FAILED probe —
                # since Build 7 the background health thread keeps
                # that cache permanently warm, so a lender killed
                # mid-generation (the drop path) leaves a fresh
                # failure cached, and the very borrow meant to
                # discover the drop read the cache and never
                # attempted the request at all: no drop recorded,
                # no fusion notice, reason "unreachable" — the
                # fusion path starved exactly like the idle path
                # did. A cached FAILURE earns one fresh probe
                # before the borrow gives up on this peer; only a
                # fresh failure keeps "unreachable". A genuinely
                # down lender pays one extra short probe. Mirrors
                # the not-lending fresh check below and the
                # busy/idle grace paths, which already re-probe
                # with force=True. (A cached SUCCESS is untouched:
                # freshness only ever ADDS an attempt, and a failed
                # fresh probe of a just-dropped lender still
                # classifies through _mesh_borrow_once.)
                status, err = mesh_probe_peer(peer, force=True)
            if status is None:
                # A failed probe — cached or fresh — never clears
                # a recorded drop. The drop is the freshest thing
                # this machine knows about the peer (Build 6's
                # manifest: warning "<name> dropped" read straight
                # after it), and a lender that died mid-answer is
                # exactly the peer whose next probes all fail: the
                # failure EXPLAINS the drop, it does not supersede
                # it. The status payload already weighs the two —
                # the drop stands until a probe SUCCEEDS (its
                # success stamp postdates the drop), a borrow is
                # served, or five minutes pass. (An earlier repair
                # popped the record here the moment a fresh probe
                # proved the peer dark; that raced the drop
                # discovery itself — the borrow meant to find the
                # drop could read a warm failed cache, force one
                # dark probe, and erase the record before the row
                # ever showed it.)
                if relay_target is None and (err or "unreachable") in ("unreachable", "timeout"):
                    # 0.2.0e Build 7 (#154): the direct road to
                    # this peer is dark at the first stage — it is
                    # the relay road's candidate destination. The
                    # relay is attempted only after the direct
                    # loop settles (direct first, always), and
                    # only when a consented carrier exists.
                    relay_target = peer
                last_reason = err or "unreachable"
                continue
            name = peer["name"]
            with MESH_LOCK:
                MESH_STATE["borrow_peer"] = name
            if not status.get("lending"):
                # Build 7 gate repair: the probe above may have
                # served a CACHED status (up to 20s old), and the
                # background health thread now keeps that cache
                # permanently warm — so a lender that stopped and
                # then re-enabled lending inside the cache window
                # would read as still-off and the borrow would
                # never even be attempted. Re-probe ONCE fresh
                # before refusing; only a fresh not-lending (or a
                # failed fresh probe) earns "lending-off". A
                # genuinely-off lender pays one extra short probe
                # on this path. Mirrors the busy-grace path below,
                # which already re-probes with force=True.
                fresh, _ferr = mesh_probe_peer(peer, force=True)
                if fresh is None or not fresh.get("lending"):
                    return None, {"servedBy": "local", "fellBack": True, "reason": "lending-off", "node": name}
                status = fresh
            outcome = _mesh_borrow_once(peer, messages)
            if outcome[0] == "ok":
                # Build .87: a served borrow is the freshest word
                # on the peer — it clears any recorded drop, so the
                # status row stops warning about a peer that is
                # demonstrably answering again.
                with MESH_LOCK:
                    MESH_STATE["peer_dropped"].pop(peer["nodeId"], None)
                mesh_ledger_append({"peer": peer["nodeId"], "direction": "borrowed", "wallMs": outcome[2], "outcome": "ok"})
                return outcome[1], {"servedBy": "mesh", "node": name, "fellBack": False, **_mesh_route_fields(peer, _fed_leg)}
            if outcome[0] == "dropped":
                # R3 FUSION: the notice is data; the caller runs a
                # FRESH local reply — two models' prose is never
                # spliced.
                return None, {
                    "servedBy": "local", "fellBack": True, "reason": "lender-dropped", "node": name,
                    "notice": name + " dropped mid-answer \u2014 this reply was generated locally instead.",
                }
            if outcome[0] == "error":
                if outcome[1] in ("unreachable", "timeout"):
                    # 0.2.0e Build 7 (#154): the road died between
                    # the probe and the ask — dark is dark, and the
                    # relay road gets its chance before the local
                    # fallback speaks. Any other error kind keeps
                    # today's behavior exactly.
                    served = _mesh_relay_fallback(peer, messages, ms, _fed_leg)
                    if served is not None:
                        return served
                return None, {"servedBy": "local", "fellBack": True, "reason": outcome[1], "node": name}
            reason = outcome[1]
            if reason == "idle":
                # Build 7 gate repair: "idle" is the ONE admission
                # refusal that carries a TIME dimension — the lender
                # accepts this borrow once it has been quiet for
                # idleAfterSeconds (default 60s). A borrower that
                # sends chats back-to-back right after a served
                # borrow would otherwise watch every follow-up fall
                # back locally as "idle" — the reason also renders
                # in the chat notice as "is not idle yet" — which is
                # indistinguishable in the payload from the request
                # never having been attempted, and it starves the
                # in-flight fusion path entirely (a borrow refused
                # here never registers on the lender). It gets the
                # busy shape, not the busy word: poll the lender's
                # probe-reported idle state for up to graceMs and
                # retry admission ONCE the moment the lender itself
                # reports idle; a refused retry keeps reason
                # "idle", and a mid-generation lender still
                # classifies through the busy/drop paths below and
                # in _mesh_borrow_once. Deliberately NO unbounded
                # wait: a lender idle only after the grace is spent
                # serves the NEXT request, exactly like busy (R5).
                deadline = time.time() + max(0, ms["borrow"]["graceMs"]) / 1000.0
                while time.time() < deadline:
                    time.sleep(0.4)
                    st2, _e2 = mesh_probe_peer(peer, force=True)
                    if st2 is not None and st2.get("idle"):
                        outcome = _mesh_borrow_once(peer, messages)
                        if outcome[0] == "ok":
                            with MESH_LOCK:
                                MESH_STATE["peer_dropped"].pop(peer["nodeId"], None)
                            mesh_ledger_append({"peer": peer["nodeId"], "direction": "borrowed", "wallMs": outcome[2], "outcome": "ok"})
                            return outcome[1], {"servedBy": "mesh", "node": name, "fellBack": False, **_mesh_route_fields(peer, _fed_leg)}
                        if outcome[0] == "dropped":
                            return None, {
                                "servedBy": "local", "fellBack": True, "reason": "lender-dropped", "node": name,
                                "notice": name + " dropped mid-answer \u2014 this reply was generated locally instead.",
                            }
                        if outcome[0] == "error":
                            return None, {"servedBy": "local", "fellBack": True, "reason": outcome[1], "node": name}
                        reason = outcome[1]
                        break
                return None, {"servedBy": "local", "fellBack": True, "reason": reason, "node": name}
            if reason == "busy":
                # R5 GRACE: poll/retry admission for up to graceMs.
                # Once local generation starts (grace spent), the
                # lender is never claimed mid-reply — it serves the
                # NEXT request.
                deadline = time.time() + max(0, ms["borrow"]["graceMs"]) / 1000.0
                while time.time() < deadline:
                    time.sleep(0.4)
                    st2, _e2 = mesh_probe_peer(peer, force=True)
                    if st2 is not None and not st2.get("busy"):
                        outcome = _mesh_borrow_once(peer, messages)
                        if outcome[0] == "ok":
                            with MESH_LOCK:
                                MESH_STATE["peer_dropped"].pop(peer["nodeId"], None)
                            mesh_ledger_append({"peer": peer["nodeId"], "direction": "borrowed", "wallMs": outcome[2], "outcome": "ok"})
                            return outcome[1], {"servedBy": "mesh", "node": name, "fellBack": False, **_mesh_route_fields(peer, _fed_leg)}
                        break
                return None, {"servedBy": "local", "fellBack": True, "reason": "busy", "node": name}
            return None, {"servedBy": "local", "fellBack": True, "reason": reason, "node": name}
        if relay_target is not None:
            # 0.2.0e Build 7 (#154): every direct road settled
            # dark — the relay road is tried last, and only for
            # the first dark destination in card order (Ring 1
            # never races lenders, on any road).
            served = _mesh_relay_fallback(relay_target, messages, ms, _fed_leg)
            if served is not None:
                return served
        return None, {"servedBy": "local", "fellBack": True, "reason": last_reason}
    finally:
        with MESH_LOCK:
            MESH_STATE["borrow_out"] = False
            MESH_STATE["borrow_peer"] = ""


def mesh_try_borrow(messages, cfg, ms, route=None):
    # The borrower's public door — _mesh_try_borrow_inner does
    # the work; this wrapper keeps the automatic road honest
    # (v0.4.2e, #231). With NO stored route the precedence
    # prefers the fedinet: when the answer nevertheless came
    # from a lower road — a Mesh peer served, or every road
    # fell back to local — the payload names the road that was
    # preferred (autoFellFrom / autoFedinet), so the line under
    # the reply records the fallback instead of letting the
    # answer pose as the first choice. An explicit route never
    # carries the mark: its roads name themselves through the
    # route fields and the route-unavailable line.
    reply, payload = _mesh_try_borrow_inner(messages, cfg, ms,
                                            route=route)
    if route is not None or not isinstance(payload, dict):
        return reply, payload
    try:
        feds = chat_route_availability(cfg)["fedinets"]
    except Exception:
        feds = []
    if not feds:
        return reply, payload
    if payload.get("servedBy") == "mesh" \
            and not payload.get("fedinet"):
        payload = dict(payload, autoFellFrom="federation",
                       autoFedinet=feds[0]["name"])
    elif payload.get("fellBack"):
        payload = dict(payload, autoFellFrom="federation",
                       autoFedinet=feds[0]["name"])
    return reply, payload


# -- Chat routes (v0.4.1e, #228) --------------------------------
# The composer's route pill is the chat's route control, and
# this store is its one source of truth — the switch gate in
# structural form, the same shape the Federation page uses: the
# pill's face renders from the bridge's answer, a flip is a
# request the bridge validates against the access the operator
# actually has, and a refusal changes nothing and says why on
# the pill itself. A route is per chat:
#   local      — model work runs on this machine; no borrow is
#                attempted at all (the old "This chat only").
#   mesh       — model work borrows over the pairwise Mesh:
#                hand-paired machines only, never a fedinet's.
#   federation — model work borrows through one fedinet: only
#                that fedinet's members are candidates, and the
#                pill wears the fedinet's name in green.
# A chat with NO stored route rides the AUTOMATIC precedence
# (v0.4.2e, #231): the fedinet first, then the Mesh, then
# local, falling back down the chain as availability changes —
# and the pill's face for such a chat is that effective
# default, computed below by the bridge, never a page-side
# guess, and marked as automatic so it can never be mistaken
# for a route the operator chose.

CHAT_ROUTES_PATH = os.path.join(ROOT, "config", "chat-routes.json")
_chat_routes_lock = threading.Lock()


def _chat_route_clean(raw):
    if not isinstance(raw, dict):
        return None
    mode = raw.get("mode")
    if mode == "local":
        return {"mode": "local"}
    if mode == "mesh":
        return {"mode": "mesh"}
    if mode == "federation":
        fid = str(raw.get("fedinetId") or "")[:80]
        if not fid:
            return None
        return {"mode": "federation", "fedinetId": fid,
                "name": str(raw.get("name") or "")[:80]}
    return None


def _chat_routes_load():
    try:
        with open(CHAT_ROUTES_PATH) as f:
            data = json.load(f)
        if isinstance(data, dict):
            out = {}
            for k, v in data.items():
                c = _chat_route_clean(v)
                if c is not None:
                    out[str(k)[:80]] = c
            return out
    except Exception:
        pass
    return {}


def chat_route_get(chat_id):
    if not chat_id:
        return None
    with _chat_routes_lock:
        return _chat_routes_load().get(str(chat_id)[:80])


def _chat_route_store(chat_id, route):
    with _chat_routes_lock:
        routes = _chat_routes_load()
        if route is None:
            routes.pop(str(chat_id)[:80], None)
        else:
            routes[str(chat_id)[:80]] = route
        try:
            os.makedirs(os.path.dirname(CHAT_ROUTES_PATH),
                        exist_ok=True)
            with open(CHAT_ROUTES_PATH, "w") as f:
                json.dump(routes, f, indent=2)
        except Exception:
            pass


def chat_route_availability(cfg):
    # Access, in the bridge's own words — the ONE computation
    # the pill, the rail, and the badges all read (v0.4.2e,
    # #234). Mesh is available when borrowing is on and at
    # least one HAND-PAIRED machine can be borrowed from (a
    # federation-MADE record is not a hand pairing; a dual-hat
    # machine — hand-paired AND a roster member — is, and
    # counts for both roads). A federation stop exists for each
    # joined fedinet with at least one borrowable member,
    # membership RECOGNIZED from the roster (peer_fedinet_id),
    # never from the stamp a record was created with: the
    # field's hand-paired gateway carried "viaFedinet": "" and
    # the federation was invisible to this computation while
    # the rail — which reads the same joined fedinets from
    # federation_settings — correctly showed it. A mode
    # without access is named Not Available by the pill and
    # refused here — the two can never disagree, because both
    # read this one computation.
    ms = mesh_settings(cfg)
    peers = []
    if ms["enabled"] and ms["borrow"]["enabled"]:
        peers = [p for p in ms["borrow"]["peers"] if p.get("canBorrow")]
    fedinets = federation_settings(cfg)
    mesh_ok = any(not _peer_federation_made(p) for p in peers)
    fed_names = {}
    for f in fedinets:
        fed_names[f["id"]] = f.get("name") or f["id"]
    feds = []
    seen = set()
    for p in peers:
        fid = peer_fedinet_id(cfg, p, fedinets)
        if fid and fid in fed_names and fid not in seen:
            seen.add(fid)
            feds.append({"id": fid, "name": fed_names[fid]})
    return {"mesh": mesh_ok, "fedinets": feds}


def chat_route_answer(cfg, chat_id):
    # Everything the pill renders from: the stored route (or
    # None), the effective face when nothing is stored — the
    # AUTOMATIC precedence (v0.4.2e, #231): the first fedinet
    # with a borrowable member, then the Mesh, then Local —
    # and the access table. "automatic" says which kind of
    # effective this is: True when it came from the precedence
    # (nothing stored), False when it is the operator's own
    # stored choice, so the pill's face can mark the one and
    # never dress it as the other. One answer, three readers
    # (face, flip, reload), so the three-way contract holds
    # by construction.
    stored = chat_route_get(chat_id)
    avail = chat_route_availability(cfg)
    effective = stored
    automatic = False
    if effective is None:
        automatic = True
        if avail["fedinets"]:
            f0 = avail["fedinets"][0]
            effective = {"mode": "federation",
                         "fedinetId": f0["id"], "name": f0["name"]}
        elif avail["mesh"]:
            effective = {"mode": "mesh"}
        else:
            effective = {"mode": "local"}
    return {"ok": True, "chatId": str(chat_id or "")[:80],
            "route": stored, "effective": effective,
            "automatic": automatic,
            "available": avail}


def chat_route_set(cfg, chat_id, raw):
    # A flip, validated. Returns (answer, None) on success — the
    # answer carries the route the bridge actually stored — or
    # (None, (reason, sentence)) on refusal, with the store
    # untouched: the pill keeps its old face and shows the
    # sentence, never a silent snap-back.
    if isinstance(raw, dict) and raw.get("mode") == "automatic":
        # v0.4.3e (#235): the return to automatic. The stored
        # route is REMOVED, not replaced with a marker value:
        # the chat goes back to standing exactly as a chat that
        # was never routed stands, and chat_route_answer
        # computes its road from the precedence and marks it
        # automatic — no stored value masquerades as the
        # default. Clearing is never refused (it needs no road
        # to be available), and the answer is read back from
        # the store after the clear, so the pill's face, the
        # store, and a post-reload read agree by construction.
        _chat_route_store(chat_id, None)
        return chat_route_answer(cfg, chat_id), None
    route = _chat_route_clean(raw)
    if route is None:
        return None, ("route-shape", "That is not a route the pill offers.")
    avail = chat_route_availability(cfg)
    if route["mode"] == "mesh" and not avail["mesh"]:
        return None, ("mesh-not-available",
                      "Mesh is not available: no paired machine can "
                      "be borrowed from right now.")
    if route["mode"] == "federation":
        match = None
        for f in avail["fedinets"]:
            if f["id"] == route["fedinetId"]:
                match = f
        if match is None:
            return None, ("federation-not-available",
                          "That federation is not available: it is "
                          "not joined here, or no member of it can "
                          "be borrowed from right now.")
        route["name"] = match["name"]
    _chat_route_store(chat_id, route)
    return chat_route_answer(cfg, chat_id), None


def mesh_probe_reason(kind):
    # 0.2.0e Build 1 (#143): a probe/borrow failure kind in full
    # words — what happened, and the cure where the product knows
    # it. One source of truth: the Mesh page's peer rows render
    # this under the short state word, and the health check
    # speaks the same sentences. Short surfaces (the Servers
    # row's warning) keep the bare kind by design.
    return {
        "fingerprint": "The machine answered, but the certificate it presented is not the one its pairing card pinned \u2014 its Mesh identity has changed since the card was made. The cure is a fresh card: import that machine's new PNG card on the Mesh page and the pin is replaced in place \u2014 the pair's history is kept.",
        "unauthorized": "The machine answered and its certificate matches its card, but it refused this machine's signature. First suspect: the two machines' clocks disagree \u2014 a Mesh signature lives for 120 seconds, so a clock more than two minutes out reads as a forgery. Check the clock on both machines; if they agree, re-pair from fresh cards.",
        "tls": "Something answered at the address, but the secure (TLS) handshake failed \u2014 whatever is listening there is not speaking the Mesh door's TLS, or the connection was cut mid-handshake.",
        "timeout": "The machine did not answer in time \u2014 it may be asleep, off, or on another network.",
        "unreachable": "Nothing answered at any address on its card \u2014 the machine is off, asleep, on another network, or its Mesh door is not listening.",
    }.get(kind, "")


def mesh_status_payload(cfg, via_dhd=False):
    # The /api/status mesh section and the Settings card's state:
    # metadata only — state, peers, balances. No content, ever.
    # Build .87: the payload also carries the status column's
    # at-a-glance verdict — a summary "state" + short "warning"
    # string, and per peer a "status" word + "lastProbeError" —
    # all read from state the bridge already tracks (the probe
    # cache, the recorded drops, the lend/borrow flags). Names and
    # state words only: a peer's addresses never enter this
    # payload, exactly as they never render anywhere.
    # Build 6 (#86): "viaDhd" says whether THE REQUEST this
    # payload answers arrived through the DHD proxy surface (the
    # X-Apex-Via: dhd header the proxy stamps). It is a property
    # of the request, not of the machine, so the callers that
    # answer an operator's view pass it in; everyone else gets
    # False. The dashboard reads it to render the Lend switch
    # locked on the remote view — the in-person turn-on lock
    # itself is enforced at the settings handler, unchanged.
    ms = mesh_settings(cfg)
    balances = mesh_balances()
    peers = []
    now = time.time()
    for p in ms["borrow"]["peers"]:
        b = balances.get(p["nodeId"], {"lentSec": 0.0, "borrowedSec": 0.0, "lastTs": 0})
        cache = MESH_STATE["probe_cache"].get(p["nodeId"])
        reachable = None
        probe = None
        probe_err = ""
        probe_stamp = 0.0
        probe_age = None
        if cache:
            # Build 7 (#87): the last probe's word is DURABLE —
            # it no longer expires after 60s. A failed check
            # stays "unreachable", with its error kind and its
            # age, until a newer probe replaces it; reachable is
            # None (the roster's "not checked yet") only when no
            # probe has ever completed for this peer. Freshness
            # now comes from the background health cadence, not
            # from forgetting.
            reachable = cache[1] is not None
            probe = cache[1] if isinstance(cache[1], dict) else None
            probe_err = str(cache[2] or "")
            probe_stamp = cache[0]
            # The age reads the cache's fourth element — the
            # last SUCCESSFUL probe's stamp (the first failed
            # probe's, for a peer never reached) — never the
            # latest attempt's, so it grows through an outage.
            probe_age = max(0, int(now - (cache[3] if len(cache) > 3 and cache[3] else cache[0])))
        caps = p.get("capabilities")
        if probe is not None and probe.get("capabilities"):
            live_caps = _mesh_caps_sanitize(probe.get("capabilities"))
            if live_caps is not None:
                caps = live_caps
        # The peer's one-word state: a recorded drop that no later
        # healthy probe has superseded outranks the probe cache —
        # it is the freshest thing this machine knows about the
        # peer, and it is exactly what the operator needs warned
        # about. Drops age out after five minutes so a stale one
        # cannot haunt the row forever. Build 7 gate repair: the
        # superseding stamp is the last SUCCESSFUL probe's (the
        # cache's fourth element), never the latest ATTEMPT's —
        # a failed health probe landing after a drop (the lender
        # is down; every cadence tick fails) advanced the attempt
        # stamp past the drop and flipped the row's warning from
        # "<name> dropped" to "<name> unreachable", hiding the
        # drop behind the outage it caused. A peer that ANSWERS
        # again still supersedes the drop at once, exactly as
        # before; a peer that merely gets probed and fails does
        # not.
        drop_stamp = MESH_STATE["peer_dropped"].get(p["nodeId"], 0.0)
        ok_stamp = cache[3] if cache and len(cache) > 3 and cache[3] else 0.0
        if drop_stamp and now - drop_stamp < 300 and drop_stamp >= ok_stamp:
            peer_status = "dropped"
        elif reachable is True:
            peer_status = "reachable"
        elif reachable is False:
            peer_status = "unreachable"
        else:
            peer_status = "unknown"
        _tun = p.get("tunnel") or {}
        if _tun.get("enabled"):
            _tst = mesh_tunnel_state(p["nodeId"])
            _tun_word = _tst.get("state") or "down"
            _tun_sentence = _tst.get("sentence") or (
                "The backup tunnel to " + p["name"] + " is not running.")
        else:
            _tun_word, _tun_sentence = "off", ""
        peers.append({
            "name": p["name"],
            "nodeId": p["nodeId"],
            "short": p["nodeId"][:12],
            "canBorrow": p["canBorrow"],
            "reachable": reachable,
            "status": peer_status,
            "lastProbeError": probe_err if reachable is False else "",
            # 0.2.0e Build 1 (#143): the same failure in full
            # words, for the Mesh page's peer rows (see
            # mesh_probe_reason).
            "lastProbeReason": mesh_probe_reason(probe_err) if reachable is False else "",
            # Build 7 (#87): seconds since the peer was last
            # successfully checked — or since its first check
            # when it has never been reached — or null when it
            # has never been probed at all. The roster renders
            # "(last checked 42s ago)" from it beside a failed
            # state, and through an outage it keeps growing:
            # failed attempts refresh the reason, not the age.
            "lastProbeAgeSec": probe_age,
            "balanceSec": round(b["lentSec"] - b["borrowedSec"], 1),
            "lastTs": b["lastTs"],
            "capabilities": caps,
            "probeLending": (bool(probe.get("lending")) if probe is not None else None),
            "probeBusy": (bool(probe.get("busy")) if probe is not None else None),
            "probeIdle": (bool(probe.get("idle")) if probe is not None else None),
            # Build 4 (#154): the peer's declared public
            # endpoint — the one address of a peer any surface
            # may show, because its operator typed it for
            # publication on the card. Harvested addresses stay
            # plumbing and never enter this payload.
            "publicEndpoint": p.get("publicEndpoint") or "",
            # Build 4 (#178): the peer's backup route as this
            # machine holds it — its config, whether its card
            # carried a tunnel key, whether THIS machine has
            # authorized that key, and the tunnel's live state
            # in the supervisor's words.
            "tunnel": _tun,
            "hasTunnelKey": bool(p.get("tunnelPublicKey")),
            "tunnelKeyAuthorized": mesh_tunnel_key_authorized(
                p.get("tunnelPublicKey") or ""),
            "tunnelState": _tun_word,
            "tunnelSentence": _tun_sentence,
            # Build 6 (#143): the walked backup-route path's
            # prefill — the SSH door this peer is KNOWN to have,
            # derived from the endpoint its operator declared on
            # its card (host, and its port when the endpoint's
            # shape carries one a tunnel can use; the SSH port is
            # otherwise the standing 227 the Mesh page's field
            # already defaults to). The operator's own typed
            # values always win; this only fills a blank.
            "tunnelSuggestion": _mesh_tunnel_suggestion(p),
            # 0.2.0e Build 7 (#154): this machine's relay consent
            # ticks as stored on this peer's record — the three
            # lists the peer card's Relay section renders its
            # switches from, so the page, the saved config, and a
            # reload all read the same truth.
            "relayCarryTo": list(p.get("relayCarryTo") or []),
            "relayAcceptFrom": list(p.get("relayAcceptFrom") or []),
            "relayWillCarryTo": list(p.get("relayWillCarryTo") or []),
        })
    # The summary the Servers row renders (build .87): one state
    # word for the dot, one SHORT warning for the label — the
    # first problem peer by name, never an address.
    state = "off"
    warning = ""
    if ms["enabled"]:
        dropped_peer = next((p for p in peers if p["status"] == "dropped"), None)
        unreach_peer = next((p for p in peers if p["status"] == "unreachable"), None)
        if not peers:
            state = "no-peers"
        elif dropped_peer is not None:
            state = "warning"
            warning = str(dropped_peer["name"]) + " dropped"
        elif unreach_peer is not None:
            state = "warning"
            warning = str(unreach_peer["name"]) + " unreachable"
        elif MESH_STATE["borrow_out"]:
            state = "borrowing"
        elif MESH_STATE["borrowed_conn"] is not None:
            state = "lending"
        elif any(p["status"] == "reachable" for p in peers):
            state = "reachable"
        else:
            state = "on"
    return {
        "enabled": ms["enabled"],
        "role": ms["role"],
        "nodeName": ms["nodeName"],
        "nodeId": mesh_node_id() or ms["nodeId"],
        "publicEndpoint": ms["publicEndpoint"],
        "listenPorts": list(ms["listenPorts"]),
        "tunnelPublicKey": mesh_tunnel_public_key() if ms["enabled"] else "",
        "borrowEnabled": ms["borrow"]["enabled"],
        "graceMs": ms["borrow"]["graceMs"],
        "lendEnabled": ms["lend"]["enabled"],
        "lend": ms["lend"],
        "ledgerInBackup": ms["ledgerInBackup"],
        "certNoticeSeen": ms["certNoticeSeen"],
        "borrowing": bool(MESH_STATE["borrow_out"]),
        "lendingNow": MESH_STATE["borrowed_conn"] is not None,
        "state": state,
        "warning": warning,
        # 0.2.0e Build 8 (#154; ruling 2026-11-13): the headline
        # graduates from census to state — "Mesh Online" while
        # this node keeps any fedinet; "" while it keeps none,
        # and every surface keeps its census words.
        "headline": federation_headline(cfg),
        "viaDhd": bool(via_dhd),
        "capabilities": mesh_capabilities(cfg),
        "peers": peers,
    }


# -- Mesh health check (0.2.0e Build 1, #143) -----------------------
# The field diagnosis of #143 prescribed two decisive reads, and
# they were never run: both needed a terminal on two machines at
# once. (1) Compare the fingerprint this node HOLDS for a peer
# against the fingerprint that peer actually PRESENTS. (2) On a
# node whose door answers as a stranger, find out WHAT is
# serving the door. This check runs both, from inside the
# product, on the operator's word from the Mesh page.
# Diagnostics only: it pairs nothing, pins nothing, routes
# nothing \u2014 it reads, compares, and reports in plain words,
# with the actual fingerprints shown side by side.

def _mesh_tls_presented(host, port, timeout):
    # One TLS handshake with verification off (the pin is the
    # authentication, and the CALLER compares it). Returns
    # (stage, fingerprint): "ok" with the presented certificate's
    # SHA-256, or the stage that failed \u2014 "connect", "tls",
    # "timeout".
    ctx = ssl._create_unverified_context()
    try:
        raw = socket.create_connection((host, int(port)), timeout=timeout)
    except (socket.timeout, TimeoutError):
        return "timeout", ""
    except OSError:
        return "connect", ""
    try:
        raw.settimeout(timeout)
        tls = ctx.wrap_socket(raw, server_hostname=host)
    except (socket.timeout, TimeoutError):
        try:
            raw.close()
        except Exception:
            pass
        return "timeout", ""
    except Exception:
        # ssl.SSLError, or the connection cut mid-handshake: the
        # failure IS the handshake.
        try:
            raw.close()
        except Exception:
            pass
        return "tls", ""
    try:
        der = tls.getpeercert(binary_form=True) or b""
    except Exception:
        der = b""
    try:
        tls.close()
    except Exception:
        pass
    if not der:
        return "tls", ""
    return "ok", hashlib.sha256(der).hexdigest()


def _mesh_bridge_processes():
    # Every process on this machine whose command line runs a
    # bridge.py \u2014 pid, and whether it is THIS process. Linux
    # /proc only; anything unreadable contributes nothing.
    out = []
    try:
        names = os.listdir("/proc")
    except Exception:
        return out
    for name in names:
        if not name.isdigit():
            continue
        try:
            with open("/proc/" + name + "/cmdline", "rb") as f:
                raw = f.read()
        except Exception:
            continue
        parts = [q.decode("utf-8", "replace") for q in raw.split(b"\x00") if q]
        if any(q.endswith("bridge.py") for q in parts):
            out.append({"pid": int(name), "self": int(name) == os.getpid()})
    out.sort(key=lambda b: b["pid"])
    return out


def _mesh_listener_owner(port):
    # The pid owning the LISTEN socket on the Mesh port, from
    # /proc/net/tcp* socket inodes matched against /proc/*/fd.
    # None when no listener is found, or /proc will not say.
    inodes = set()
    for path in ("/proc/net/tcp", "/proc/net/tcp6"):
        try:
            with open(path) as f:
                lines = f.read().splitlines()[1:]
        except Exception:
            continue
        for ln in lines:
            f = ln.split()
            if len(f) > 9 and f[3] == "0A":
                try:
                    if int(f[1].rsplit(":", 1)[1], 16) == int(port):
                        inodes.add(f[9])
                except Exception:
                    pass
    if not inodes:
        return None
    try:
        names = os.listdir("/proc")
    except Exception:
        return None
    for name in names:
        if not name.isdigit():
            continue
        fd_dir = "/proc/" + name + "/fd"
        try:
            fds = os.listdir(fd_dir)
        except Exception:
            continue
        for fd in fds:
            try:
                target = os.readlink(os.path.join(fd_dir, fd))
            except Exception:
                continue
            if target.startswith("socket:[") and target[8:-1] in inodes:
                return int(name)
    return None


def _mesh_fp_short(fp):
    return (fp[:12] + "\u2026") if fp else ""


def _mesh_own_addresses():
    # Every address that reaches THIS machine: loopback, every
    # interface address (physical and virtual alike — the door
    # binds 0.0.0.0, so any of them lands on it), and the
    # default route's address. The health sweep judges peer
    # entries against this set (Build 4, #143).
    addrs = {"127.0.0.1", "::1", "localhost"}
    try:
        for ip in _interface_ipv4s():
            addrs.add(ip)
    except Exception:
        pass
    try:
        ip = lan_ipv4()
        if ip:
            addrs.add(ip)
    except Exception:
        pass
    return addrs


def _mesh_is_own_door(host, port, own_port, own_addrs):
    # An entry points at this machine's own Mesh door when the
    # host is one of this machine's own addresses AND the port
    # is this machine's Mesh port. Both halves matter: loopback
    # at a DIFFERENT port (127.0.0.1:8766, an SSH forward to a
    # peer's door) is a road, not the own door, and is dialed
    # like any other entry — the field's working off-LAN road
    # depends on exactly that distinction.
    try:
        return int(port) == int(own_port) and str(host) in own_addrs
    except Exception:
        return False


def mesh_health_check(cfg):
    # The check itself. Shape: {ranAt, port, enabled, self,
    # processes, peers, summary} \u2014 the Mesh page renders it
    # nearly verbatim; every verdict carries its sentence, and a
    # peer's addresses never enter the payload (stages only),
    # exactly as they never render anywhere.
    ms = mesh_settings(cfg)
    port = ms["port"]
    out = {"ranAt": int(time.time()), "port": port, "ports": list(ms["listenPorts"]),
           "enabled": ms["enabled"],
           "self": {}, "processes": {}, "peers": [], "summary": ""}
    if not ms["enabled"]:
        out["self"] = {"verdict": "off", "stage": "",
                       "diskFingerprint": "", "presentedFingerprint": "",
                       "verdictText": "Mesh is off on this machine \u2014 its door is closed by choice, so there is nothing to check. Turn Mesh on and run the check again."}
        out["processes"] = {"bridges": [], "count": 0, "listenerOwnerPid": None,
                            "listenerIsThisBridge": None, "verdictText": ""}
        out["summary"] = "Mesh is off \u2014 the check has nothing to read."
        out["relay"] = []
        return out
    # This machine: the disk certificate vs the one the door
    # actually presents. A difference here is the duplicate-
    # bridge signature: something else is serving the door.
    disk_fp = mesh_cert_fingerprint()
    stage, presented_fp = _mesh_tls_presented("127.0.0.1", port, 3)
    self_sec = {"diskFingerprint": disk_fp, "presentedFingerprint": presented_fp,
                "stage": stage}
    if stage != "ok":
        self_sec["verdict"] = "not-listening"
        self_sec["verdictText"] = ("This machine's Mesh door did not answer at 127.0.0.1:" + str(port)
            + " (the attempt failed at the " + stage + " stage). The bridge answering this dashboard "
            + "is not serving the door \u2014 if another bridge owns it, the processes check below names the owner.")
    elif presented_fp == disk_fp and disk_fp:
        self_sec["verdict"] = "match"
        self_sec["verdictText"] = ("The certificate this machine's Mesh door presents is the one on its disk "
            + "\u2014 this machine is exactly who its pairing cards say it is.")
    else:
        self_sec["verdict"] = "mismatch"
        self_sec["verdictText"] = ("The door answered with a DIFFERENT certificate than the one on this machine's disk "
            + "(" + _mesh_fp_short(disk_fp) + " on disk, " + _mesh_fp_short(presented_fp) + " presented). Something else is "
            + "serving this machine's Mesh door \u2014 most likely a second bridge running with a different identity. "
            + "Every peer that paired this machine will refuse the door until the presenting certificate is the card's. "
            + "The processes check below is the suspect list.")
    out["self"] = self_sec
    # Processes: how many bridges, and whose door it is.
    bridges = _mesh_bridge_processes()
    owner = _mesh_listener_owner(port)
    door_ports = []
    for dp in ms["listenPorts"]:
        dp_owner = owner if dp == port else _mesh_listener_owner(dp)
        door_ports.append({"port": dp, "listenerOwnerPid": dp_owner,
                           "listenerIsThisBridge": (dp_owner == os.getpid()) if dp_owner is not None else None})
    proc_sec = {"bridges": bridges, "count": len(bridges),
                "listenerOwnerPid": owner,
                "listenerIsThisBridge": (owner == os.getpid()) if owner is not None else None,
                "ports": door_ports}
    bits = []
    if len(bridges) == 0:
        bits.append("No process running a bridge.py was found on this machine \u2014 the dashboard's bridge may be running inside another program, or /proc would not say.")
    elif len(bridges) == 1:
        bits.append("One bridge process is running on this machine (pid " + str(bridges[0]["pid"]) + ").")
    else:
        bits.append(str(len(bridges)) + " bridge processes are running on this machine (pids "
                    + ", ".join(str(b["pid"]) for b in bridges) + "). Two bridges cannot share the Mesh door: "
                    + "the second one's door never opens, and if the one that grabbed the door presents a different "
                    + "certificate than this machine's card, every peer sees a stranger. This is the prime suspect "
                    + "when both machines swear their certificates match.")
    for dp_info in door_ports:
        dp = dp_info["port"]
        dp_owner = dp_info["listenerOwnerPid"]
        if dp_owner is None:
            bits.append("No listener was found on Mesh door port " + str(dp) + ".")
        elif dp_owner == os.getpid():
            bits.append("The Mesh door's listener (port " + str(dp) + ") belongs to this bridge (pid " + str(dp_owner) + ").")
        else:
            bits.append("The Mesh door's listener (port " + str(dp) + ") belongs to pid " + str(dp_owner) + " \u2014 not this bridge (pid " + str(os.getpid()) + ").")
    proc_sec["verdictText"] = " ".join(bits)
    out["processes"] = proc_sec
    # Peers: the pin this machine HOLDS vs the certificate the
    # peer actually PRESENTS \u2014 then, when the identity matches,
    # the signature stage.
    for peer in ms["borrow"]["peers"]:
        ent = {"name": peer["name"], "nodeId": peer["nodeId"], "short": peer["nodeId"][:12],
               "pinnedFingerprint": peer["certFingerprint"], "presentedFingerprint": "",
               "stage": "", "verdict": "", "authOk": None, "verdictText": "",
               "route": "",
               # Build 6 (#143): the in-place heal, when this
               # record's addresses are all dead ends — offered
               # on the same card that names the problem.
               # (v0.3.0e — Build 6, #227): the second heal —
               # a federation-marked record missing roads its
               # verified gateway card carries.
               "repair": mesh_record_repair(peer) or mesh_card_refresh_repair(cfg, peer)}
        presented_other = ""
        presented_own = ""
        skipped_own = 0
        dialed = 0
        matched = False
        matched_entry = ""
        # Build 4 (#178): the sweep reads the same dial list the
        # borrow path does — the record's entries, then the
        # managed tunnel endpoint last — and remembers which
        # entry answered, so a healthy answer that arrived over
        # the backup road is named as exactly that.
        tunnel_entry = _mesh_tunnel_entry(peer)
        stages = []
        # Build 4 (#143): probe manners. An entry that is this
        # machine's own door (own address, own Mesh port) is
        # skipped without dialing — it could only ever answer as
        # this machine. An answer presenting this machine's OWN
        # certificate is named as exactly that — your own door —
        # and the sweep moves on: it is not a peer fingerprint
        # mismatch, because the peer was never reached. (The
        # field failure: a card's container-bridge address
        # terminated at the probing machine's own door, and the
        # machine's own certificate was reported as the peer's
        # "changed identity" for a week.)
        own_addrs = _mesh_own_addresses()
        for address in _mesh_effective_addresses(peer):
            host, eport = _mesh_addr_parts(address, peer["port"])
            if _mesh_is_own_door(host, eport, port, own_addrs):
                skipped_own += 1
                continue
            dialed += 1
            st, fp = _mesh_tls_presented(host, eport, 3)
            stages.append(st)
            if st == "ok":
                if fp == peer["certFingerprint"]:
                    matched = True
                    matched_entry = address
                    ent["presentedFingerprint"] = fp
                elif fp and disk_fp and fp == disk_fp:
                    if not presented_own:
                        presented_own = fp
                elif not presented_other:
                    presented_other = fp
        if matched:
            status, err = mesh_probe_peer(peer, force=True)
            if status is not None:
                ent["stage"] = "ok"
                ent["verdict"] = "match"
                ent["authOk"] = True
                ent["verdictText"] = (peer["name"] + " answers, the certificate it presents matches its card, "
                    + "and it accepts this machine's signature. This pair is healthy."
                    + (" The answer came via the backup tunnel — the SSH door carried it."
                       if tunnel_entry and matched_entry == tunnel_entry else ""))
                if tunnel_entry and matched_entry == tunnel_entry:
                    ent["route"] = "tunnel"
            elif err == "unauthorized":
                ent["stage"] = "auth"
                ent["verdict"] = "unauthorized"
                ent["authOk"] = False
                ent["verdictText"] = mesh_probe_reason("unauthorized")
            else:
                ent["stage"] = err or "unreachable"
                ent["verdict"] = err or "unreachable"
                ent["verdictText"] = ("Its certificate matches its card, but the signed check failed: "
                    + mesh_probe_reason(err or "unreachable"))
        elif presented_other:
            ent["presentedFingerprint"] = presented_other
            ent["stage"] = "fingerprint"
            ent["verdict"] = "mismatch"
            ent["verdictText"] = (peer["name"] + " answered, but the certificate it presented ("
                + _mesh_fp_short(presented_other) + ") is not the one its card pinned ("
                + _mesh_fp_short(peer["certFingerprint"]) + ") \u2014 its Mesh identity has changed since the card "
                + "was made. The cure is a fresh card: import " + peer["name"] + "'s new PNG card on this page "
                + "and the pin is replaced in place \u2014 the pair's history is kept. (And if " + peer["name"]
                + " swears its certificate never changed, run this check there: a second bridge may be serving its door.)")
        elif presented_own:
            ent["presentedFingerprint"] = presented_own
            ent["stage"] = "own-door"
            ent["verdict"] = "own-door"
            ent["verdictText"] = (peer["name"] + " did not answer \u2014 the door that answered presented this machine's "
                + "OWN certificate (" + _mesh_fp_short(presented_own) + "), which is to say the probe reached "
                + "your own door, not " + peer["name"] + "'s. An address on its card leads back to this machine "
                + "(a container bridge or a stale local address does exactly this), so the pair was never actually "
                + "asked. Nothing about " + peer["name"] + "'s identity is in doubt \u2014 it was never reached. The cure "
                + "is a fresh card minted on " + peer["name"] + " itself: cards minted now carry only addresses "
                + "that machine can be reached at.")
        elif skipped_own and not dialed:
            ent["stage"] = "own-door"
            ent["verdict"] = "own-door"
            ent["verdictText"] = ("Every address on " + peer["name"] + "'s card is this machine's own Mesh door \u2014 "
                + "its own addresses at its own port \u2014 so there is nothing on the card that could reach "
                + peer["name"] + " at all. The card was minted from the wrong machine's addresses or edited by "
                + "hand. The cure is a fresh card minted on " + peer["name"] + " itself and imported here.")
        else:
            if "tls" in stages:
                worst = "tls"
            elif stages and all(x == "timeout" for x in stages):
                worst = "timeout"
            else:
                worst = "connect"
            ent["stage"] = worst
            ent["verdict"] = "unreachable"
            ent["verdictText"] = mesh_probe_reason("tls" if worst == "tls" else "timeout" if worst == "timeout" else "unreachable")
        # Build 4 (#178): when the pair did not answer and a
        # backup route exists, the tunnel's own state rides the
        # verdict — down, not authorized, or unreachable, in the
        # supervisor's words, never as a bare code.
        if tunnel_entry and ent["verdict"] != "match":
            _ts = mesh_tunnel_state(peer["nodeId"])
            if _ts.get("sentence"):
                ent["verdictText"] += " " + _ts["sentence"]
        out["peers"].append(ent)
    # 0.2.0e Build 7 (#154): the relay roads this machine has
    # consented to — one verdict per (destination, carrier) pair
    # whose asker tick is set HERE. The verdict reads the two
    # ends this machine can actually check: its own tick (local
    # truth, by construction of the loop) and the carrier's own
    # tick (asked over the wire, with a signed probe the carrier
    # answers from its own state). The destination's acceptance
    # is the one end no probe can read — it is proven when the
    # destination answers a carriage — and the sentence says
    # exactly that rather than rounding the road up to whole.
    out["relay"] = []
    for dest in ms["borrow"]["peers"]:
        for relay in ms["borrow"]["peers"]:
            if relay["nodeId"] == dest["nodeId"]:
                continue
            if dest["nodeId"] not in (relay.get("relayCarryTo") or []):
                continue
            word = _mesh_relay_probe(relay, dest["nodeId"])
            if word == "consent":
                verdict = "consented"
                text = (relay["name"] + " will carry this machine's envelopes to "
                        + dest["name"] + " — the relay road stands consented at both "
                        "ends this check can read. " + dest["name"] + "'s own acceptance "
                        "is proven when it answers a carriage.")
            elif word == "no-consent":
                verdict = "no-consent"
                text = ("The relay road to " + dest["name"] + " via " + relay["name"]
                        + " is consented here, but " + relay["name"] + " has not ticked "
                        "that it will carry between this machine and " + dest["name"]
                        + " — the tick is " + relay["name"] + "'s to give, on its own dashboard.")
            elif word == "dest-unknown":
                verdict = "dest-unknown"
                text = (relay["name"] + " is not paired with " + dest["name"]
                        + ", so it cannot carry this machine's envelopes there — "
                        "carriage needs the carrier paired with both ends.")
            elif word == "unauthorized":
                verdict = "unauthorized"
                text = (relay["name"] + " answered the relay probe but refused this "
                        "machine's signature — the pairing or the clocks disagree; "
                        "the peer check above names the same failure in full.")
            else:
                verdict = "dark"
                text = (relay["name"] + " did not answer the relay probe, so the "
                        "relay road to " + dest["name"] + " is dark at its first hop.")
            out["relay"].append({"to": dest["nodeId"], "toName": dest["name"],
                                 "via": relay["nodeId"], "viaName": relay["name"],
                                 "verdict": verdict, "verdictText": text})
    # The summary: the first problem, or the all-clear.
    problems = []
    if self_sec["verdict"] == "mismatch":
        problems.append("this machine's door is presenting the wrong certificate")
    elif self_sec["verdict"] == "not-listening":
        problems.append("this machine's Mesh door is not answering")
    for ent in out["peers"]:
        if ent["verdict"] != "match":
            problems.append(ent["name"] + ": " + ent["verdict"])
    if problems:
        out["summary"] = "Found " + ("a problem" if len(problems) == 1 else str(len(problems)) + " problems") + " \u2014 " + "; ".join(problems) + "."
    elif out["peers"]:
        out["summary"] = "Everything the check can see is consistent: this machine presents the certificate its cards advertise, and every paired machine matches its card and accepts this machine's signature."
    else:
        out["summary"] = "This machine's own door is consistent \u2014 it presents the certificate on its disk. No machines are paired yet, so there are no peer checks to run."
    return out


# -- The door's knock ledger (0.2.0e Build 4, #143/#154) ------------
# The Mesh door can face the open internet now (a declared
# public endpoint is exactly that), so it keeps count of failed
# knocks per source: failed TLS handshakes at accept time, and
# requests that fail signature authentication. Five failures
# inside a minute close the door to that source — connections
# are dropped at accept, before any handler runs — for a
# backoff that doubles per repeat closing (60s, 120s, 240s, …
# capped at 900s). Every knock and every closing is logged to
# the bridge's stdout in words. A successful authentication
# clears the source's failure slate: a paired peer's valid
# signed traffic is never throttled, and whole requests are
# not chatty, so no legitimate federation pattern can reach
# five failures inside a minute. State is in-memory only: a
# bridge restart reopens every door, which is the honest
# failure direction for a lockout.

MESH_DOOR_KNOCK_FAILS = 5
MESH_DOOR_KNOCK_WINDOW = 60
MESH_DOOR_BLOCK_BASE = 60
MESH_DOOR_BLOCK_CAP = 900
_MESH_DOOR_LOCK = threading.Lock()
_MESH_DOOR = {}


def _mesh_door_log(msg):
    print("Navi Mesh: " + msg, flush=True)


def mesh_door_blocked(ip):
    now = time.time()
    with _MESH_DOOR_LOCK:
        st = _MESH_DOOR.get(ip)
        return bool(st and st["blockedUntil"] > now)


def mesh_door_knock(ip, kind):
    # One failed knock from a source: a handshake that failed
    # at the door, or a request its signature did not carry.
    # Returns "counted", or "blocked" when this knock closed
    # the door (or the door was already closed).
    if not ip:
        return "counted"
    now = time.time()
    with _MESH_DOOR_LOCK:
        st = _MESH_DOOR.setdefault(ip, {"fails": [], "blockedUntil": 0.0, "blocks": 0, "dropLogged": False})
        if st["blockedUntil"] > now:
            return "blocked"
        st["fails"] = [t for t in st["fails"] if now - t < MESH_DOOR_KNOCK_WINDOW]
        st["fails"].append(now)
        n = len(st["fails"])
        if n >= MESH_DOOR_KNOCK_FAILS:
            st["blocks"] += 1
            secs = min(MESH_DOOR_BLOCK_CAP, MESH_DOOR_BLOCK_BASE * (2 ** (st["blocks"] - 1)))
            st["blockedUntil"] = now + secs
            st["fails"] = []
            st["dropLogged"] = False
            _mesh_door_log("door closed to " + str(ip) + " for " + str(secs) + "s — " + str(n)
                           + " failed knocks (" + str(kind) + ") inside " + str(MESH_DOOR_KNOCK_WINDOW)
                           + "s. It reopens on its own when the backoff runs out.")
            return "blocked"
        _mesh_door_log("door knock from " + str(ip) + " (" + str(kind) + ") — " + str(n) + " of "
                       + str(MESH_DOOR_KNOCK_FAILS) + " inside the window before the door closes to it.")
        return "counted"


def mesh_door_success(ip):
    # A valid signed request: the source's slate is wiped. A
    # paired peer therefore cannot accumulate its way into a
    # closing across honest mistakes spaced by real work.
    if not ip:
        return
    with _MESH_DOOR_LOCK:
        st = _MESH_DOOR.get(ip)
        if st:
            st["fails"] = []


def mesh_door_drop(ip):
    # A connection from a closed source, dropped at accept:
    # logged once per closing, so a flood cannot flood the log.
    with _MESH_DOOR_LOCK:
        st = _MESH_DOOR.get(ip)
        if st and not st["dropLogged"]:
            st["dropLogged"] = True
            _mesh_door_log("door dropped a connection from " + str(ip) + " — the door is still closed to it.")


class _MeshDoorServer(ThreadingHTTPServer):
    # The Mesh door's server: the accept loop itself minds the
    # knock ledger. The listening socket is deliberately NOT
    # TLS-wrapped: wrapping the listener would run every
    # handshake inside accept() itself — before any hook of
    # ours, uncounted, and with no bound on how long a stalled
    # handshake may park the serve loop. Instead each accepted
    # raw socket is wrapped here, in get_request, with a
    # bounded wait: a source the door is closed to is dropped
    # before any TLS work is spent on it, a handshake that
    # fails is a counted knock, and only a completed handshake
    # ever reaches a handler thread. The accepted socket
    # returns to blocking for the handler, exactly as the
    # wrapped-listener shape it replaces behaved — the wire
    # cannot tell the difference; the ledger can.
    daemon_threads = True
    mesh_tls_context = None

    def get_request(self):
        while True:
            sock, addr = super().get_request()
            ip = addr[0] if addr else ""
            if ip and mesh_door_blocked(ip):
                mesh_door_drop(ip)
                try:
                    sock.close()
                except Exception:
                    pass
                continue
            try:
                sock.settimeout(10)
                tls = self.mesh_tls_context.wrap_socket(
                    sock, server_side=True)
                tls.settimeout(None)
            except Exception:
                if ip:
                    mesh_door_knock(ip, "handshake")
                try:
                    sock.close()
                except Exception:
                    pass
                continue
            return tls, addr


# -- The dedicated Mesh door -----------------------------------------
# Mesh ALWAYS listens on its own TLS port (build .86) — its own
# door, its own certificate, in every configuration. The DHD proxy
# refuses /mesh/* outright and the dashboard surface does not
# serve the node-facing endpoints at all: borrowers dial this
# port — the peer card's port — only. There is no plaintext Mesh
# listener.


# -- Quick Sync over the Mesh (0.2.0e Build 6, #176 phase 2) ---------
# Sync's second transport: the same sync set (chats + Case Files,
# format navi-sync/1, tombstones included) rides whole signed Mesh
# requests to a paired peer instead of the backup repo. The
# exchange is two-way in ONE request — the borrower sends its set,
# the lender converges it and answers with its own set, the
# borrower converges that — so one answered door settles both
# machines. Convergence is sync_apply's, unchanged: the transport
# differs, the semantics (including deletion tombstones) do not.
# Serving requires the pairing (mesh_authenticate) AND a proven
# shared account (v0.6.4e, #323). The pre-gate law read pairing
# itself as consent, with one refusal that fired only when BOTH
# machines knew their operator's GitHub login and they differed
# — a check that failed open on unknown identity, and did: the
# operator's chats, Case Files, notes and day book converged
# onto a paired peer's brand-new, never-signed-in seat (the
# field report that filed #323). Pairing is consent between
# operators — for borrow, chat, fleet. Sync is one operator's
# own data moving between that operator's own seats, so its
# door demands positive proof: a signed account handshake
# (/mesh/sync/hello) states each side's login, and a set crosses
# only when both logins are present and EQUAL. Unknown is a
# refusal, never a pass — in both directions, at every door.

def _mesh_sync_account_verdict(body):
    # "" when the requester's stated account is proven equal to
    # this machine's own; otherwise the refusal's name —
    # "operator-unknown" when either side is unstated,
    # "different-operator" when both are stated and differ.
    mine = str((github_auth() or {}).get("login") or "")
    theirs = str((body or {}).get("operatorLogin") or "")
    if not mine or not theirs:
        return "operator-unknown"
    if mine.lower() != theirs.lower():
        return "different-operator"
    return ""


def _mesh_sync_set_wire():
    collected, manifest, _index = sync_collect()
    files = {}
    for path, blob in collected.items():
        files[path] = base64.b64encode(blob).decode("ascii")
    return files, manifest


def _mesh_sync_decode(payload):
    # A sync set off the wire, {path: bytes}, or None when the
    # payload is not shaped like one. Paths may only live under
    # sync/ and may not climb — a set is data, never a route.
    files_in = (payload or {}).get("files")
    if not isinstance(files_in, dict):
        return None
    out = {}
    for path, b64 in files_in.items():
        p = str(path)
        if not p.startswith("sync/") or ".." in p.split("/"):
            return None
        try:
            out[p] = base64.b64decode(str(b64), validate=True)
        except Exception:
            return None
    return out


def _mesh_sync_admit(handler, body):
    # The borrow handler's admission shape, shared by the two
    # sync routes: parse the body, then authenticate the peer.
    # Returns (data, peer) or (None, (code, payload)).
    try:
        data = json.loads((body or b"").decode("utf-8", "replace"))
    except Exception:
        return None, (400, {"ok": False, "reason": "bad-request"})
    if not isinstance(data, dict):
        return None, (400, {"ok": False, "reason": "bad-request"})
    peer = mesh_authenticate(handler, body, str(data.get("requestId") or ""))
    if peer is None:
        return None, (401, {"ok": False, "reason": "unauthorized"})
    verdict = _mesh_sync_account_verdict(data)
    if verdict:
        # v0.6.4e (#323): the refusal is ledgered like an
        # exchange is — a refused door is a fact of the road,
        # named with its reason, never a silent nothing.
        mesh_ledger_append({"peer": peer["nodeId"], "direction": "sync",
                            "outcome": "refused-" + verdict})
        return None, (200, {"ok": False, "reason": verdict,
                            "account": str((github_auth() or {}).get("login") or "")})
    return (data, peer), None


def mesh_handle_sync_hello(handler, body):
    # POST /mesh/sync/hello — the account handshake (v0.6.4e,
    # #323): a paired peer states its account, this machine
    # states its own, and NO set data rides in either direction.
    # The initiator verifies the answer before any set byte is
    # offered; this door is how it learns what the responder
    # claims to be. Authentication is the pairing's, unchanged.
    try:
        data = json.loads((body or b"").decode("utf-8", "replace"))
    except Exception:
        return 400, {"ok": False, "reason": "bad-request"}
    if not isinstance(data, dict):
        return 400, {"ok": False, "reason": "bad-request"}
    peer = mesh_authenticate(handler, body, str(data.get("requestId") or ""))
    if peer is None:
        return 401, {"ok": False, "reason": "unauthorized"}
    mesh_ledger_append({"peer": peer["nodeId"], "direction": "sync",
                        "outcome": "hello"})
    return 200, {"ok": True,
                 "account": str((github_auth() or {}).get("login") or ""),
                 "nodeId": mesh_node_id() or ""}


def mesh_handle_sync_pull(handler, body):
    # POST /mesh/sync/pull — a paired peer asks for this
    # machine's sync set (a trigger round's pull). Answers with
    # the set; applies nothing.
    t0 = time.time()
    admitted, refusal = _mesh_sync_admit(handler, body)
    if admitted is None:
        return refusal
    _data, peer = admitted
    files, manifest = _mesh_sync_set_wire()
    mesh_ledger_append({"peer": peer["nodeId"], "direction": "sync",
                        "wallMs": int((time.time() - t0) * 1000),
                        "outcome": "pull"})
    return 200, {"ok": True, "files": files, "manifest": manifest}


def mesh_handle_sync_exchange(handler, body):
    # POST /mesh/sync/exchange — the full round in one request:
    # converge the borrower's set, then answer with this
    # machine's set for the borrower to converge. The sync lock
    # serializes against this machine's own rounds — a collision
    # answers "busy" and the borrower falls back to the repo,
    # never waits.
    t0 = time.time()
    admitted, refusal = _mesh_sync_admit(handler, body)
    if admitted is None:
        return refusal
    data, peer = admitted
    theirs = _mesh_sync_decode(data)
    if theirs is None:
        return 200, {"ok": False, "reason": "bad-set"}
    if not _SYNC_LOCK.acquire(False):
        return 200, {"ok": False, "reason": "busy"}
    # v0.5.1e (#257): the lender's applies are sync-driven, not
    # writes — the round-depth guard covers them too, so
    # converging a borrower's set never notes a push of its own
    # (an echo round here would collide with the borrower's
    # next round at this very door). What the apply records —
    # tombstones included — rides this seat's next round.
    if _SYNC_ROUND["depth"] == 0:
        _SYNC_ROUND["thread"] = threading.get_ident()
    _SYNC_ROUND["depth"] += 1
    try:
        applied = sync_apply(load_config(), {
            "files": theirs,
            "manifest": data.get("manifest") if isinstance(data.get("manifest"), dict) else {},
        })
    finally:
        _SYNC_ROUND["depth"] = max(0, _SYNC_ROUND["depth"] - 1)
        if _SYNC_ROUND["depth"] == 0:
            _SYNC_ROUND["thread"] = None
        _SYNC_LOCK.release()
    files, manifest = _mesh_sync_set_wire()
    mesh_ledger_append({"peer": peer["nodeId"], "direction": "sync",
                        "wallMs": int((time.time() - t0) * 1000),
                        "outcome": "exchange"})
    return 200, {"ok": True, "files": files,
                 "manifest": manifest, "applied": applied}


class MeshHandler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def log_message(self, *args):
        pass

    def _send(self, code, payload):
        body = json.dumps(payload).encode()
        try:
            self.send_response(code)
            self.send_header("Content-Type", "application/json")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            self.wfile.write(body)
        except Exception:
            pass

    def _body(self):
        length = int(self.headers.get("Content-Length") or 0)
        return self.rfile.read(length) if length > 0 else b""

    def do_GET(self):
        if self.path.split("?")[0] == "/mesh/status":
            code, payload = mesh_handle_status(self, b"")
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/intel-digest":
            code, payload = mesh_handle_intel_digest(self, b"")
            self._send(code, payload)
        else:
            self._send(404, {"ok": False, "reason": "not-found"})

    def do_POST(self):
        if self.path.split("?")[0] == "/mesh/relay":
            code, payload = mesh_handle_relay(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/chat":
            # FediChat (v0.5.3e, #267): the room's door —
            # roster-authenticated inside, like the roster door.
            code, payload = mesh_handle_chat(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/roster":
            code, payload = mesh_handle_roster(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/sync/hello":
            code, payload = mesh_handle_sync_hello(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/sync/pull":
            code, payload = mesh_handle_sync_pull(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/sync/exchange":
            code, payload = mesh_handle_sync_exchange(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/seraph-report":
            code, payload = mesh_handle_seraph_report(self, self._body())
            self._send(code, payload)
        elif self.path.split("?")[0] == "/mesh/borrow":
            code, payload = mesh_handle_borrow(self, self._body())
            if code == "drop":
                # Build .87 (gate 4, borrower half): a handler-level
                # connection.close() on this HTTP/1.1 keep-alive TLS
                # connection does NOT deliver a prompt EOF to a
                # borrower blocked in getresponse() — the borrow
                # strands for minutes instead of completing. Hard-
                # close at the SOCKET level: shutdown(SHUT_RDWR)
                # first, so the termination lands on the wire at
                # once, then close. The borrower reads termination
                # as the drop signal and its fusion fallback speaks.
                self.close_connection = True
                try:
                    self.connection.shutdown(socket.SHUT_RDWR)
                except Exception:
                    pass
                try:
                    self.connection.close()
                except Exception:
                    pass
                return
            self._send(code, payload)
        else:
            self._send(404, {"ok": False, "reason": "not-found"})



# -- Managed backup route (0.2.0e Build 4, #178) -------------------
# The SSH door stays a Mesh road, but the bridge keeps the road
# itself now: for a peer whose backup route is on, the bridge
# spawns and supervises its own `ssh -N -L` forward — the peer's
# Mesh door, carried through the peer's SSH door, presented on a
# bridge-owned loopback port — and that endpoint joins the peer's
# dial list LAST, computed at dial time, never written into the
# stored record. Direct entries are always tried first; a dead
# tunnel costs nothing beyond the ordinary per-entry timeouts.
# The key is a dedicated ed25519 pair: the public half rides this
# node's pairing card, and the lender authorizes it once, in
# person, as a forwarding-only line in ~/.ssh/authorized_keys.
# The private half lives at 0600 under config/mesh and never
# rides a backup — backups collect named files, and key files
# are never collected (the node keypair's custody, unchanged).
# Hand-run tunnels and hand-edited 127.0.0.1 records keep
# working untouched beside all of this: a record entry is a
# record entry, however it got there.

MESH_TUNNELS = {}
MESH_TUNNEL_LOCK = threading.Lock()
_MESH_TUNNEL_BACKOFFS = (5, 15, 60)


def mesh_ensure_tunnel_key():
    # (ok, error-sentence). Generated with ssh-keygen — the same
    # OpenSSH tooling the supervisor runs — so the private half
    # is in exactly the format ssh reads.
    if os.path.isfile(MESH_TUNNEL_KEY_PATH) and \
            os.path.isfile(MESH_TUNNEL_PUB_PATH):
        return True, ""
    try:
        os.makedirs(MESH_DIR, exist_ok=True)
        r = subprocess.run(
            ["ssh-keygen", "-t", "ed25519", "-N", "",
             "-C", "navi-mesh-tunnel", "-f", MESH_TUNNEL_KEY_PATH],
            capture_output=True, timeout=30)
        if r.returncode != 0 or not os.path.isfile(MESH_TUNNEL_KEY_PATH):
            return False, ("This machine could not create its Mesh tunnel "
                           "key — ssh-keygen did not answer. The managed "
                           "backup route needs the OpenSSH client tools.")
        os.chmod(MESH_TUNNEL_KEY_PATH, 0o600)
        return True, ""
    except Exception:
        return False, ("This machine could not create its Mesh tunnel "
                       "key — ssh-keygen is not available. The managed "
                       "backup route needs the OpenSSH client tools.")


def mesh_tunnel_public_key():
    # This node's tunnel public key line, or "" — generated on
    # first need (pairing-card mint, Mesh page), like the node
    # identity beside it.
    if not os.path.isfile(MESH_TUNNEL_PUB_PATH):
        ok, _err = mesh_ensure_tunnel_key()
        if not ok:
            return ""
    return read_file(MESH_TUNNEL_PUB_PATH).strip()


def _tunnel_key_blob(pub_line):
    parts = str(pub_line or "").split()
    if len(parts) >= 2 and parts[0].startswith("ssh-"):
        return parts[1]
    return ""


def mesh_tunnel_key_authorized(pub_line):
    # Whether THIS machine's authorized_keys already carries the
    # given tunnel key (the lender-side read the Mesh page shows).
    blob = _tunnel_key_blob(pub_line)
    if not blob:
        return False
    ak = os.path.join(os.path.expanduser("~"), ".ssh", "authorized_keys")
    try:
        with open(ak) as f:
            lines = f.read().splitlines()
    except Exception:
        return False
    return any(blob in ln.split() for ln in lines
               if not ln.strip().startswith("#"))


def mesh_tunnel_authorize(pub_line, node_id, action, door_port):
    # (ok, sentence). The lender-side act, in person only (the
    # handler refuses it over DHD, like arming lending): the line
    # appended lets the peer's tunnel key do exactly one thing —
    # forward to this machine's Mesh door on loopback. No shell,
    # no agent, no X11. Idempotent; revoke removes the key's
    # lines. A missing .ssh or authorized_keys is created with
    # the permissions sshd insists on.
    parts = str(pub_line or "").split()
    blob = _tunnel_key_blob(pub_line)
    if not blob:
        return False, ("That machine's card carries no tunnel key — "
                       "there is nothing to authorize.")
    ssh_dir = os.path.join(os.path.expanduser("~"), ".ssh")
    ak = os.path.join(ssh_dir, "authorized_keys")
    lines = []
    if os.path.isfile(ak):
        try:
            with open(ak) as f:
                lines = f.read().splitlines()
        except Exception:
            lines = []
    if action == "revoke":
        kept = [ln for ln in lines if blob not in ln.split()]
        if len(kept) == len(lines):
            return True, ("That tunnel key was not in this machine's "
                          "authorized keys — nothing to revoke.")
        try:
            with open(ak, "w") as f:
                f.write("\n".join(kept) + ("\n" if kept else ""))
            os.chmod(ak, 0o600)
        except Exception:
            return False, "This machine could not rewrite its authorized keys file."
        return True, ("Revoked: that tunnel key no longer opens "
                      "anything on this machine.")
    if any(blob in ln.split() for ln in lines
           if not ln.strip().startswith("#")):
        return True, ("That tunnel key is already authorized on this "
                      "machine — the line was already there, unchanged.")
    try:
        os.makedirs(ssh_dir, mode=0o700, exist_ok=True)
        try:
            os.chmod(ssh_dir, 0o700)
        except Exception:
            pass
        opts = ('permitopen="127.0.0.1:' + str(door_port) + '",no-pty,'
                "no-agent-forwarding,no-X11-forwarding")
        lines.append(opts + " " + parts[0] + " " + blob +
                     " navi-mesh-tunnel:" + str(node_id or "")[:12])
        with open(ak, "w") as f:
            f.write("\n".join(lines) + "\n")
        os.chmod(ak, 0o600)
    except Exception:
        return False, "This machine could not write its authorized keys file."
    return True, ("Authorized: that tunnel key can now forward to this "
                  "machine's Mesh door — and nothing else on this machine.")


def _mesh_tunnel_suggestion(peer):
    # Build 6 (#143): what the walked backup-route path can
    # prefill for this peer — the SSH host, taken from the host
    # of the public endpoint the peer's operator declared on its
    # card (the machine's public name; its SSH door lives at the
    # same name in every deployment the field has). The endpoint's
    # PORT is the Mesh door's, never the SSH door's, so it is not
    # suggested — the field's standing 227 default stands.
    ep = str(peer.get("publicEndpoint") or "").strip()
    if not ep:
        return {"sshHost": ""}
    host, _port = _mesh_addr_parts(ep, 0)
    return {"sshHost": host or ""}


def _mesh_tunnel_entry(peer):
    # The peer's tunnel endpoint as a dial entry, or "" — present
    # only while the route is on and a local port is allocated.
    tun = peer.get("tunnel") or {}
    if tun.get("enabled") and tun.get("localPort"):
        return "127.0.0.1:" + str(tun["localPort"])
    return ""

