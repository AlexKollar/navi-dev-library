# Operator guides (docs/ in the shipped tree)

## docs/MESH.md

Source: `docs/MESH.md` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# Apex | Mesh — your machines, one pool of inference

Apex | Mesh lets your own machines share the work of answering. A modest laptop assembles a chat request the way it always does — persona, Vault, knowledge packs, conversation — and a stronger machine you own writes the reply on its own model. The routing is **whole-request**: one machine answers, start to finish. There is no split answer and no streaming between machines in Ring 1.

Mesh is off until you turn it on, and it works between machines **you** pair by hand. Nothing about it is tied to a specific person, machine name, or domain: every name and address comes from your own settings and the cards you exchange.

## Pairing — the card is the handshake

Each machine has a **Mesh card** — a PNG picture whose face a person can read and whose inside carries the card data: its name, its node id, its public key, the fingerprint of the certificate its Mesh door serves, the addresses it can be reached at, and a **capabilities block** — an honest self-report of the machine's hardware class, GPU or CPU, RAM, model, speed class, and the caps it lends under, so a borrower can see what it is borrowing from before it asks.

1. On the first machine, open the **Mesh page** (Mesh in the left menu's Apex | Systems group), turn Mesh on, and give the machine a name.
2. Choose **Download PNG card** and carry the picture file to the second machine.
3. On the second machine, turn Mesh on, then drop the picture onto the **Paired machines** area — or use **Upload a PNG card** — and the peer is added.
4. Do the same in the other direction — each machine needs the other's card.

Pairing is the consent act: a paired machine is a trusted machine, and a paired machine's signed requests are accepted on that basis alone. Either side can unpair at any time by removing the card. The **can borrow** toggle on a peer is the borrowing machine's own choice about that peer: untick it there and that machine will not borrow from the peer, while staying paired — it does not gate what the peer accepts.

**Which addresses a card carries.** A card's addresses are harvested by *interface*, not by address range: only the addresses bound to the machine's physical Ethernet/Wi-Fi interfaces are minted — never loopback, never a container bridge or its tails, never a VM bridge, tunnel, or VPN adapter. A 10.x address on the physical network card is a legitimate LAN address and rides; a 192.168.x address on a container bridge does not. If a machine has no plausible physical-LAN address at all — and no declared public endpoint, below — the card **refuses to mint**, and says why in plain words, rather than minting a card that strands the pair. Cards from earlier builds carried every address the machine had, junk included; re-minting a machine's card under this build and re-importing it is the cure for a pair whose addresses name bridges and ghosts instead of the machine.

**A port for every address.** An address entry may carry its own port, as `host:port`; an entry without one inherits the record's port, exactly as every card before this build worked. One record can therefore speak LAN at home and a public door away — the borrower tries the entries in order and the first that presents the card's pinned certificate serves.

**The public endpoint.** If a machine's Mesh door can be reached from beyond its own networks — because you forwarded a port to it — declare that door on its Mesh page in the **Public endpoint** field, as host:port. You type it; Apex never harvests it. It rides the machine's card beside its LAN addresses and lands in each peer's record as a `host:port` entry of its own, so the pair works at home and away off the one record — no tunnel, no hand edits. Changing it means re-sharing the card, the same as a new name. Leave it empty and the card carries the LAN addresses and nothing else.

**The card is a picture — the only way it travels.** **Download PNG card** on the Mesh page saves the card as a .png. Its face is the card a person reads — the machine's name, its node identity, and the offer in words: its capabilities, the lending limits it is configured with, and whether it lends when idle or even while busy. The whole card rides inside the picture, in the same data-in-picture format the chips use (a PNG text chunk carrying the card), so Apex keeps one cartridge language across the product. On the other machine, drop the picture onto the Mesh page's **Paired machines** area, or use **Upload a PNG card** there, and the peer is added — a PNG exported by one machine imports into another with every field intact. Earlier builds also offered the card as copyable text; that path is gone — there is nothing to paste anymore, and a text card saved from an earlier build can no longer be used. Machines already paired are unaffected: pairing was the consent, and the record it made never depended on how the card travelled.

**Send the picture as a file.** The payload survives only if the file's bytes arrive untouched. Put the .png inside a **.zip** before sending it through a chat app: Discord and similar apps recompress the images people upload and strip out the extra data chunks, leaving a picture that looks like the card and pairs nothing. The same goes for any channel that re-encodes images — send it as a file attachment, a zip, a USB stick, anything byte-exact. When a payload-less picture is imported, the Mesh page says so in plain words — *this image has no readable Mesh card data; it may have been recompressed by a chat app; send the original file inside a .zip instead* — and a damaged or truncated picture is refused just as specifically, never silently ignored.

## Borrowing — automatic, with off switches

Once Mesh is on and a lender is paired and reachable, borrowing happens by itself: a chat request goes to the first reachable lender, and the reply arrives with a small pill under it — the Hexlink glyph and **via <machine name>** in Mesh cyan.

Three off switches, from widest to narrowest:

- The **Mesh master switch** turns the whole thing off.
- The **Borrowing switch** stops this machine from borrowing while leaving it able to lend.
- **This chat only: keep it on this machine**, beside the composer, holds one chat session home — no borrow is attempted for it at all.

If the lender is **busy**, your machine waits a short grace (2 seconds by default, changeable on the Mesh page) for it to free up, then answers locally; a lender that frees up during the grace can still take that request, but once your machine starts answering, the lender is never claimed mid-reply — it serves the next request. If a lender **drops mid-answer**, you see the notice — *"<machine> dropped mid-answer — this reply was generated locally instead"* — and your machine writes a fresh reply of its own. Two machines' prose is never spliced together.

CLI mode, chips, and every built-in route never borrow. Only model answers do.

## Lending — automatic when idle, off means off

Lending can only be turned **on** from the lending machine's own dashboard, in person, at that machine. It cannot be armed through the DHD door. Turning it off works from anywhere you are signed in — and **Stop lending now** ends lending and aborts a borrowed generation already running, mid-flight, at once.

Open the Mesh page through the DHD door and the **Lend this machine** switch shows the lock itself: while lending is off it renders locked — dimmed, not clickable — with the reason in plain words directly under the switch, because lending can only be turned on in person. While lending is on, the switch stays live on the remote view, since turning lending off works from anywhere; a note under it says turning it back on will require being at the machine. The lock is the bridge's, not the page's: the bridge refuses a lending turn-on that arrives through the DHD door regardless of what any client renders.

A lender serves only inside the limits you set:

- **Idle only** (on by default): the machine must have been quiet for 60 seconds, and no local generation may be running.
- **Share cap**: borrowed work may take at most a set share of any rolling hour — **25%** as shipped.
- **Generation caps**: at most a set number of borrowed generations an hour — **20** as shipped — and a set number for any one machine — **10** as shipped.

The three caps are yours to set, on the lending machine itself: they sit in the **Lending** section of its Mesh page, beside the Caps line that shows the current values, and they take effect as soon as you save — no restart. Set them for the machine it is; a strong server can take far more than the shipped defaults. Your card advertises the caps you set, so a paired machine sees the real offer before it asks.

**Idle only** has an override in the same section: the **Lend only when idle** switch, on as shipped. Turn it off and the machine lends without waiting for a quiet minute — borrowed generations run alongside your own work, and your own replies may slow down. The caps still apply, and lending still serves one borrowed generation at a time: a request that arrives mid-generation waits for the next slot, exactly as before. Your card states the posture either way — *lends when idle* or *lends even while busy*.

Your own work always wins: a local request takes the next slot ahead of any new borrowed work, though it never kills a borrowed answer already running. A request whose assembled context exceeds the lender's context setting is refused whole — never silently truncated.

## Privacy — what travels, what never does

**What travels** when you borrow: the assembled prompt for that one reply — your Soul and Memory text, any Vault or knowledge-pack material the question pulled in, and the recent conversation — sent encrypted to the lending machine, whose model reads it to write the reply. The lender keeps none of it: no prompt or reply text is written to disk on the lending machine, ever. But that machine's operator owns the machine it runs on. **Only borrow from machines you trust with the conversation.** The Mesh page shows this in plain words the first time borrowing is enabled.

**What never travels**: your files and data stores, your credentials and access codes, and command-line access. Mesh carries chat requests and replies, nothing else. Mesh keys and dashboard codes are separate credentials that share nothing in either direction: a guest signed in through DHD can never reach Mesh, and a paired Mesh peer gains no dashboard access. A peer's harvested network addresses likewise stay in the plumbing — they are used to dial the machine and are never shown in the roster or any other surface. The one deliberate exception is the **public endpoint**: an address the operator typed for publication, and the only address any surface shows.

Every Mesh request is signed with the borrowing machine's Ed25519 node key and checked against the key on its paired card; stale or unsigned requests are refused with no detail given. The connection itself is pinned to the certificate fingerprint on the card — if the machine answering presents a different certificate, the borrower refuses and says so in plain words.

## The ledger and balances

Each machine keeps a small ledger of Mesh work: who, which direction, how long, how it ended — metadata only, never content. The Mesh page's **roster** shows each paired machine's **balance** in seconds lent minus seconds borrowed, alongside its capability line and live status (reachable, lending, busy, idle, unreachable). That status is kept current by the bridge itself: while Mesh is on, every paired peer is checked in the background about every thirty seconds — and immediately when a peer is paired and when Mesh is turned on — so a pair reaches "checked" on its own, with no chat sent. A failed check stays on the roster with its reason and how long it has been since the peer last answered — *unreachable — unauthorized (last checked 42s ago)* — until a newer check replaces it; the age keeps growing while the peer stays down, because failed checks refresh the reason, not the clock. "not checked yet" appears only before a peer's first check has ever completed. The roster shows names, capabilities, status, and balances — a machine's harvested addresses are plumbing for the transport and never appear on any screen; the endpoint a peer declared on its card is the one address shown, marked as declared. Balances are a record, not a rule: Ring 1 has no give-to-get enforcement. The ledger stays out of GitHub backups unless you tick **Include the Mesh ledger in backups** (off by default); the node keypair never rides a backup either way. The pairing-card pictures are a separate matter: **Mesh pairing cards** (Settings → GitHub & Backup, its own essential, ticked like the rest) carries this machine's own card as a PNG — minted fresh from the live card as the backup runs — and a re-drawn PNG for every paired peer, so a restored machine lands the pictures in `config/mesh/cards/` and re-pairs the peers through the same card check a hand import passes. Your own picture is kept as a file; restoring it never overwrites the identity of the machine doing the restoring.

## Getting there — the dedicated door

Mesh always travels through its **own dedicated TLS door**, whether or not DHD is on. The door listens on the Mesh port — **8765, always** — and on every additional number in its **listen ports** list (the Mesh settings ship **8443** and **2475** alongside 8765; add or remove numbers there, all above 1024). It is one door on several numbers: the same protocol, the same certificate, the same fingerprint on every port, so a peer record entry like `host:8443` or `host:2475` reaches the same door as `host:8765` — which is what lets a pair use an egress-friendly number from networks that strangle unusual ports, and what a forwarded public endpoint usually names. The certificate it serves is the Mesh certificate under `config/mesh/`, and that is the fingerprint the paired card pins — on every port, in every configuration, so a card stays valid when DHD comes and goes. Mesh never rides the DHD door: the DHD proxy refuses `/mesh/*` outright, and the dashboard itself serves only the dashboard-local Mesh controls (the card view, Stop lending, the settings API) behind the access-code gate — the node-facing status and borrow endpoints live only on the dedicated port. There is no plaintext Mesh listener. Ring 1 does no NAT traversal, no relay, and no lender racing: the first reachable lender serves, and a home pair on one network is the shape it is built and tested for. Reaching a lender across the internet is the operator's road to build: forward a public port to the machine's Mesh door (any public port — an egress-friendly one like 8443 serves from networks that strangle unusual ports), then declare that host:port as the machine's **public endpoint** and re-share its card. The pair then works from home and away off the one record, with no tunnel and no hand-edited addresses.

Because a declared endpoint puts the Mesh door on the open internet, the door minds its knocks: it counts failed handshakes and failed signature checks per source, and five failures inside a minute close the door to that source — connections dropped at accept, on **every** door port at once, because the knock ledger belongs to the door and not to any one number — for a backoff that doubles per repeat closing (60 seconds, doubling, capped at 900) and reopens on its own. Every knock and every closing is logged in words on the bridge. A valid signed request clears its source's slate, so a paired machine's ordinary traffic is never throttled; the ledger is memory only, and a bridge restart reopens every door.

**The backup route.** A pair can also keep a second, quieter road: an SSH tunnel that Apex itself maintains, per peer, so the SSH door (port 227, as the operator's gateway forwards it) can carry the Mesh when the direct road cannot be used — from a network that lets almost nothing out, say. Turn it on for a peer on the Mesh page (**Backup route**, with the SSH host, port, and user) and the bridge keeps the tunnel alive on its own: it allocates a stable local port for the peer, supervises the tunnel process, restarts it with backoff when it drops, and puts it back after a bridge restart. No terminal ever opens and no password is ever used. The tunnel runs on a **dedicated keypair the bridge generates for this purpose alone** — its public half rides this machine's pairing card and shows on its Mesh page as a copyable line; the private half never leaves the machine and never rides a backup, exactly like the node keypair. On the *lending* machine, that key is admitted once, **in person, from its own dashboard** — the roster's **Authorize tunnel key** writes a single-purpose line into `~/.ssh/authorized_keys` that can open a forward to the Mesh door and nothing else: no shell, no agent, no other port. Revoking removes exactly that line. At dial time the tunnel's local endpoint is appended **last** to the peer's addresses — computed, never written into the stored record — so direct addresses are always tried first and a dead tunnel costs nothing beyond the ordinary per-address timeouts. When the answering address *is* the tunnel, the health check says so — *via the backup tunnel* — and when the tunnel is down its state is a sentence of its own: down, key not authorized, or host unreachable. A tunnel you ran by hand keeps working untouched alongside; the managed route is the same road with the bridge holding it open.

## When the Mesh goes dark — the health check

A Mesh failure used to report a single word — *unreachable — fingerprint* — and the rest was archaeology. Now the failure surfaces speak in sentences. A peer's roster row carries the full reason under its state: what answered, what did not match, and the cure where the product knows it. A changed Mesh identity is cured by importing that machine's fresh PNG card — the pin is replaced in place and the pair's ledger history is kept. A refused signature names its first suspect — the two machines' clocks — because a Mesh signature lives for 120 seconds, so a clock more than two minutes out reads as a forgery. A borrow that falls back names the stage it failed at — connect, TLS, fingerprint, signature, or timeout — in the chat notice. A Threat Intelligence digest the collector refuses is likewise refused out loud, with the check that refused it named on the Center's Mesh source line.

The **Mesh health check**, at the foot of the Mesh page, runs the two decisive reads of any dark pair on the spot. It compares the certificate on this machine's disk with the one its Mesh door actually presents — a difference means something else is serving the door, usually a second bridge, and the check's process census counts the bridges running and names the door's owner — on every one of the door's ports, so the door's own extra numbers never read as extra processes. For every paired machine it holds the fingerprint this machine pinned from its card up against the certificate that machine presents right now, and then runs the signed check, so identity and signature are judged separately. Every verdict is a sentence, with the actual fingerprints side by side. The check changes nothing: no pairing, pin, or route is touched — it reads and reports.

The check also knows its own door. An entry on a peer's card that is this machine's own Mesh door — its own address at its own port — is skipped without dialing, and if an address answers presenting this machine's OWN certificate, the verdict says exactly that — **your own door** — instead of reporting the machine's own identity as the peer's changed one. (That false mismatch, a container-bridge address terminating at the probing machine's own door, is what kept a healthy pair "broken" for a week.) Loopback at a *different* port — an SSH forward to the peer's door — is a road, not the own door, and is dialed like any other entry.

## Repairs in place, and the walked route (Build 6)

The health check only reported until Build 6; now it heals what it can, where it stands. When a peer's record carries **only dead-end addresses** — loopback in any spelling, the unspecified address, a link-local, or the default Docker bridge gateway (172.17.0.1, the address that cost the field a week) — the peer's card says so in a sentence: the record points nowhere, so the pair has never actually been asked, and no verdict about the peer's identity can be drawn from dialing it. Two heals are offered on that same card: **Import a fresh card** (the button takes you to the pairing import; the fresh card replaces the record's addresses in place) and **Clear the dead addresses** (the junk entries are stripped from the stored record now; the pairing, its pin, its tunnel route, and its history are kept — import the fresh card when you have it). A record holding at least one plausible address is never "repaired" — that is an edit, and edits arrive as fresh cards. Cards minted since Build 4 never carry junk, so this is a cure for records already on disk.

The **backup route** is a walked path now, in three presses: flip the route's switch to arm it, fill whichever field is still blank, press **Save route**. The SSH host comes prefilled from the public endpoint the peer declared on its card, when it declared one. The card also says the one thing no button can do, in plain words: the peer must authorize this machine's tunnel key **on its own dashboard, in person**, before the tunnel can open — authorization is never given from over the wire, and the bridge refuses it through the DHD door regardless of what any screen offers.

And the first time an operator pairs, the Mesh page says the quiet part **before any trust is pinned**: the door's certificate is self-signed, no authority vouches for it, and the card you were handed *is* the trust — importing it pins that machine's fingerprint here, and a door presenting a different certificate is named, never quietly trusted. The notice rides with the first pairing and is not shown again on that machine.

## Quick Sync rides the Mesh (Build 6, #176 phase 2)

Quick Sync has two transports now. When a paired peer's road is lit, the round rides **the Mesh itself**: one whole signed Mesh request carries this machine's sync set (chats + Case Files, tombstones included) to the peer, the peer converges it and answers with its own set in the same exchange, and this machine converges that — one answered door settles both machines, and the sync card names the route (*Mesh — <that machine>*). Trigger pulls (app start, DHD session start) take the same road first. When no Mesh route answers, the round falls back to the backup-repo transport, unchanged, and says so. The convergence semantics — newest wins, diverged threads kept whole as siblings, deletion tombstones honored — are the same code over both transports; a deletion made on one machine cannot be resurrected over either road. Two guards ride the Mesh route: the sync set is size-capped (past the cap, the repo carries the round), and when both machines know their operator's GitHub login and the logins **differ**, nothing crosses — the sets are different people's work, and the pairing is not a merge.

## Relay — the third road (Build 7)

Roads fail in threes: the direct addresses, the declared public endpoint, the backup tunnel — and sometimes all three are dark while a third paired machine can see both ends. That machine can **carry** the ask. The borrow is wrapped, exactly as signed for the lender, in a second signed envelope addressed to the carrier; the carrier checks three consents, forwards the inner request byte-for-byte to the lender's door, and returns the sealed answer the same way. It sees envelope metadata only — who asks, who is asked, how many bytes, what stage it ended at — and its ledger records exactly that and nothing more.

Consent is **three ticks, one on each machine**, each given on that machine's own dashboard in the **Relay** section of the carrier's peer card: the asker ticks that the carrier may carry its envelopes to the lender; the carrier ticks that it will carry between the pair; the lender ticks that the carrier may carry envelopes to it from the asker. Any missing tick refuses the carriage **by name** — the refusal says which machine has not consented — and unticking any one stops the next carriage, because every machine re-reads its consent on every request. Consent is given in person, like lending: through the DHD door a tick can be unticked from anywhere, but never ticked on.

The road is **one hop, hard**: a relayed envelope is never re-relayed. Direct is always tried first — the relay is the road of last resort — chat answers ride it, and since Build 8 a fedinet's roster sync rides it too, under the same ticks and the destination's own consent check; every other class stays direct-only. When an answer comes back carried, the reply's pill names the machine that **answered** and a line under it names the road: *via relay <machine>*. When the relay road fails, the local fallback keeps the stage's name — *no-consent*, *relay-dark*, *dest-dark*, *dest-refused* — in the same sentence style as every other failure. The health check reports a **relay verdict** for every consented road: it asks the carrier, with a signed probe that carries nothing, whether the carrier's own tick is in place and whether the destination is paired there at all, and it is honest about the one end no probe can read — the destination's own acceptance is proven when it answers a carriage.

## Federation — fedinets on the Mesh (Build 8)

Everything above is the pairwise Mesh: two machines, one card
each, trust by ceremony. Build 8 grows it into **fedinets** —
named sets of nodes whose gateway keeps a **signed roster**
(Node numbers are roster facts), admitted by **green gateway
cards** (admission + introduction only), and lent to under
**deep trust** when the operator's per-fedinet toggle is on:
the roster auto-pairs and auto-lends, refined by the
new-member tick, per-member untrust, borrow caps, and owner
preemption. Roster sync rides this door — direct, via relay,
or by member gossip — and a stale roster degrades its fedinet
to pairing-only with the sentence on the **Federation page**,
where the whole thing is administered (its own page, its own
audit log, and DHD tiers with the Operator Badge step-up for
manage acts and the full crown ceremony — Badge + named-action
confirm + audit — for enabling trust, founding, and tunnel-key
authorization). While a node keeps any fedinet, the Mesh status
headline is the state: **Mesh Online**. The full account lives
in `docs/FEDERATION.md`.

## Troubleshooting

- **"Certificate fingerprint does not match."** The machine that answered is not presenting the certificate on its card — the card is stale, or something else is answering at that address. Download a fresh PNG card from that machine's Mesh page and re-add the peer.
- **The health check says "your own door."** An address on the peer's card led back to this machine — classically a container-bridge or stale local address harvested onto an older card — and the door that answered was this machine's own. The peer was never reached and its identity is not in doubt. Mint a fresh card on the *peer* machine (cards minted now carry only reachable addresses) and import it here.
- **The card download refuses, saying the machine has no address its card can honestly carry.** Every interface the machine can see is a loopback, a container bridge, or a virtual adapter, and no public endpoint is declared. Connect the machine to a real network — Ethernet or Wi-Fi — or declare a public endpoint on the Mesh page, and the card can be minted.
- **Replies keep saying "answered locally" with a busy / cap / idle note.** The lender is doing its job: it is mid-answer, over its caps for the hour, or not idle yet. The note names which. Wait, or raise the caps on the lender.
- **"<machine> dropped mid-answer."** The lender went away during a reply — its operator hit Stop lending now, or the machine or its bridge stopped. Your reply was written locally, whole. Check the lender.
- **A peer shows unreachable.** The roster says why, and when it was last checked. *unauthorized* means the other machine does not hold this machine's card — pairing is the consent, so re-exchange cards. *fingerprint* means the machine answering is not presenting the certificate on its card. Anything else means it could not be reached at all: its Mesh is off, its bridge is stopped, or the addresses on its card no longer reach it. Checks retry on their own about every thirty seconds, and the roster updates when one succeeds.
- **"This image has no readable Mesh card data."** The PNG card arrived without its payload — almost always because a chat app recompressed it on the way. Resend the original card file inside a .zip, or by any channel that moves files byte-for-byte, and import that.
- **The status row says a peer dropped, but chats keep being answered locally.** A peer that was recorded as dropped keeps that word on its roster row while it stays down — a check that finds it still down does not replace the word, because the drop is the freshest thing this machine knows about it. The warning never gates borrowing: the next chat re-checks the peer before giving up on it, and borrows normally the moment it answers. If the machine is back, give it one health-check cycle (about thirty seconds) and the warning clears on its own.


## docs/DHD.md

Source: `docs/DHD.md` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# DHD — Apex from anywhere

DHD — Dial-Home Device, the way home — is Apex's public door. Apex | Node [local] lets a phone on your home network open Apex; DHD extends the **same dashboard** to a genuinely public address, so you can reach the Apex on your own machine from anywhere — work, the road, anywhere with a browser.

Everything is Apex's own work, assembled and managed by Apex itself. There is no Tailscale, no tunnel or proxy service, and no Certbot:

- A **reverse proxy Apex runs itself** listens on ports 80 and 443 and forwards to the bridge on `127.0.0.1:8675`. In DHD mode the bridge never binds publicly — the proxy is the only public face.
- **TLS from the first click.** With no domain, Apex generates a **self-signed certificate** and the door opens on it. With a domain, Apex's **built-in ACME client** obtains a real Let's Encrypt certificate and renews it on its own.
- The **same access code** as the home-network door guards DHD — one code, both doors. Since 0.2.0e Build 2 the code **knocks** rather than opens: a device presenting it names itself and waits for an approval (the first device approves itself), and each approved device holds its own 30-day session — see *Trusted devices* below. Nothing is served through the proxy before the code, and the public door throttles wrong codes: they slow down, and five in a row lock the door for five minutes.

DHD has its own page, reached from the menu's **Apex | Systems** group (its Node and DHD entries both open it), and the status column's **Servers** group shows it beside Llamafile and Apex | Node [local], each row with its own pulsing dot and its own dropdown of details. Directly under Navi's status, bracketed by separators, sits the update line: a green **Up to date**, or a yellow **Update available** when a check found one — that one line is the whole entry.

The DHD page is the one house for Apex's server controls: the **Apex | Node** [local] controls — the home-network door's on/off choice, its live state, and the access code — sit on the DHD page under the DHD options, so both ways of reaching Apex are managed in one place. Settings → System keeps only a pointer to them. The relocation is display-only: same config keys, same API paths, same code.

## Turning it on — one click

1. Open the **DHD** page from the menu's Apex | Systems group.
2. Flip the **master switch**. That is the click.
3. Apex does the rest in order: it generates the certificate, brings its proxy up on 80/443 in front of the bridge, and runs a local reachability check — the proxy must answer on 443 from this machine ("answered on this machine"; it proves the local half only). The page then reports the live state, the LAN address under its own label, and the public address when there is one.
4. Once the public path is set up (router forwards in place, and your domain live if you use one), open the public address from anywhere, type the **access code** (shown on the DHD page, on this machine only) and name the device — the first device at the door lets itself in, and a device that knocks later waits on its own screen until you approve it from a trusted session — and you are in: a full session, Updater included. A remote session can run the one-click update end to end, exactly like sitting at the machine.

If a step fails, the page names the step in plain words — the proxy could not take its ports, the certificate could not be prepared, the self-check did not answer — and the previous state is left completely untouched. Nothing ever half-opens.

One Linux note: ports below 1024 are reserved, so if the proxy step names them, grant the bridge the low-port capability once (`sudo setcap 'cap_net_bind_service=+ep'` on the python3 binary it runs with) and flip the switch again.

## Trusted devices — the code knocks (0.2.0e Build 2)

**The flow.** A device that has never been let in opens either door, types the access code, and gives itself a name. If it is the **first device ever** at this door, it approves itself and walks in — the bootstrap is deliberate: a one-operator house must never be locked out of its own door waiting for an approver who does not exist yet. Every device after that becomes a **pending device**: its screen waits, and it appears under **Waiting at the door** in the Door list — in **Settings → System → Door** and on the **DHD page** — where any session that is already trusted can **Approve** or **Deny** it. Approval hands that device its own session token; the waiting page notices on its own and walks through. A wait lasts 30 minutes, then the knock expires and the device knocks again.

**The session.** An approved device's session is its own — not a hash of the shared code: a random token, a **30-day** life, and a sliding renewal: using the door while the session is valid renews it, so a device in daily use never hits its 30 days mid-shift. The Door list shows every trusted device with its name, first seen, last seen, and expiry.

**Revoke and regenerate are different tools now.** **Revoke** removes one device: its token dies at once and it is back at the gate on its next knock — nobody else's session notices. **Regenerating the access code** stops new knocks with the old code (and retires the old shared-code cookie), but devices already approved keep their own tokens; revoke those one by one if you want them out. Before Build 2, changing the code was the only way to shut anyone out, and it shut everyone out; that is the problem per-device sessions exist to end.

**What did not change.** The public door still throttles wrong codes (a progressive delay; five in a row locks it for five minutes). DHD still forces the bridge onto loopback — the proxy stays the only public face. And no remote session, trusted device or not, can turn Mesh lending on: lending is armed in person, from the lending machine's own dashboard, exactly as before.

## Prerequisites, installs, and operation cards

**The prerequisites ship with Apex.** Everything the DHD stack needs — the OpenSSL binary, a writable certificate home under `config/dhd/`, the proxy component (which is part of the bridge itself) — is installed by the normal Apex installer, on every distro family, whether or not DHD is ever turned on. An update installs them too: the Updater runs the full installer (never a file-copy shortcut), and its step ladder and update log show **DHD prerequisites** as its own named step. The added weight is small and honest: OpenSSL is a roughly 1 MB package that is usually already present as a dependency of tools Apex already needs (when it is present, the step costs milliseconds and downloads nothing), the certificate home is one empty directory, and the low-port capability grant is file metadata, not data. The certificate home is also on the installer's keep-list: a `--fresh` reinstall preserves `config/dhd/` — DHD state and a real ACME certificate survive it, never wiped and forced to re-issue.

**The safety net.** If a prerequisite later goes missing — removed by hand, a partial system install — the DHD page lists it under **Prerequisites** and the enable offers an **Install** action right there: one click installs the missing piece through the same distro-aware machinery the installer uses (apt, dnf, or pacman; on immutable systems like Bazzite the page names the host-image route honestly instead of layering a package), shows its progress on the operation card, and then continues the enable on its own. A missing dependency is never a dead-end refusal; the only hard failure is the install itself failing.

**When a fix has to be yours, the commands are exact.** If an install fails, or any step in the enable ladder has a manual remedy, the failure state shows the literal command lines — complete and correct for your detected distro family — in one copyable block with a **Copy commands** button. Paste them into Apex chat (CLI mode runs them) or into a terminal.

**Operations live in chat as cards.** Ask in chat to turn DHD on and the thread shows a live operation card — prerequisites, certificate, proxy, reachability — each step marked pending, running, waiting, done, or failed, with its output expandable inline and, on failure, the exact manual commands inline with their copy button. The card is driven entirely by the bridge's own deterministic step state: **no model generation happens while an operation runs** — on home hardware the work itself is the load, and Navi stays silent until it finishes. The finished card stays in the thread as the record, the DHD page shows the same operation, and restarting the bridge mid-enable reconciles the card with the live state. Nothing on a card depends on opening an external file, link, or handler; everything it needs is inside the card.

## Addresses — domain, public IP, and LAN (build .76)

DHD works with **three different addresses**, and the page never mixes them up:

- **Your domain** — the public address once it is live: you set an **A record** for the name at your DNS host pointing at your home's public IP, your router forwards **443 and 80** to this machine, and Apex's ACME client issues the name's real certificate. Only then — name resolving to this machine, certificate live — does Apex say **outside reach verified**, and only then does the status dropdown say "from anywhere."
- **Your public IP (WAN)** — the address the internet sees for your home. Apex detects it with a small HTTPS echo lookup (cached, and it never blocks anything). With no domain, this is the public address the page shows — marked **outside reach unverified**, because Apex cannot prove your router forwards reach it until the path is actually proven. If the detected address is itself private or a carrier-NAT address (100.64.0.0/10), the page says plainly that this machine sits **behind another router/NAT** instead of presenting the address as directly reachable. If detection fails, the page says it **could not detect** the address — never a guess.
- **Your LAN address** — an address like `192.168.1.x`. It works **only on this network** and lives in its own labeled home on the DHD page and in the status dropdown: **On this network (LAN)**. It is never shown under a public label.

**What the state lines mean.** *On — listening, outside reach unverified* means the proxy is up and answering on this machine, but the outside path is not proven yet; the page names the next step in plain words (forward 443 + 80 on the router to this machine, or add your domain). The enable ladder's last step is a **local check** — "answered on this machine" — and it is labeled that way; it never claims anything about the internet. **Reachable anywhere** appears only for a verified domain setup.

**Ports 80/443 and Apex's own proxy.** When the listener on 80 or 443 is Apex's own DHD proxy, the availability rows read **"In use by Apex's DHD proxy — this door working"** — a healthy state, not an occupancy problem — and re-enabling with the proxy already running is a success, never "port is not free." That error is reserved for a *different* program holding the port.

## The parts you may still handle by hand

One click covers everything on the machine. Two things live outside it, in your router and your domain, and they are yours:

- **Router port forwarding.** Forward ports **80** and **443** (TCP) on your router to the machine Apex runs on. Without the forward, the internet cannot reach the proxy at all, and in domain mode port 80 is also how the certificate proof arrives — the DHD page will tell you plainly: "port 80 isn't reaching this machine — check the router forward."
- **Pointing your domain here.** At your domain's DNS host, set an **A record** for the name (for example `apex.example.com`) to your home's public IP address. **Dynamic IP caveat:** most home connections change address from time to time; when yours does, the name stops resolving here until the record follows. A dynamic-DNS hostname avoids the manual step — you can put a DDNS name in the domain field just as well.

The domain is **optional and addable at any time**: leave the field empty as long as you like. Adding one later changes nothing about the door except its certificate — the page holds the domain as "certificate pending" until the ACME step completes, and the self-signed door keeps answering meanwhile.

## Certificates — the warning, the real one, and renewal

**The self-signed warning is expected.** Until a domain is added, Apex serves a certificate it signed itself. Your browser cannot verify a self-signed certificate with any public authority, so it warns — on some browsers, loudly. That warning means "no authority vouches for this certificate," not "this connection is unencrypted": the traffic is still TLS-encrypted, and the access code is still the door's lock. It is the normal state of a no-domain DHD.

**With a domain, the warning goes away.** Apex's built-in ACME client talks to Let's Encrypt directly: it places an order, proves you control the domain by serving the HTTP-01 challenge through its own proxy on port 80, finalizes, downloads the certificate, installs it, and reloads the proxy. No Certbot, no external ACME tool.

**Renewal is automatic.** Let's Encrypt certificates live 90 days. Apex checks the expiry when the bridge starts and once a day after that, and renews on its own when 30 days or fewer remain — same ACME flow, no operator action. The DHD page always shows the current expiry.

## Turning it off

Flip the master switch on the DHD page back off. The proxy stops, ports 80/443 close, and Apex answers only on this machine and — if Apex | Node is on — your home network, exactly as before DHD. Your domain and certificate stay saved on the machine, so turning DHD back on later is the same one click.

Turning DHD off does not touch Apex | Node, the access code, or anything else. If you also forwarded ports on your router, you can leave the forwards in place (nothing is listening) or remove them — either is fine.


## docs/FEDERATION.md

Source: `docs/FEDERATION.md` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# Apex | Federation — fedinets, rosters, and deep trust

Build 8 of 0.2.0e takes the Mesh from a set of hand-made pairs to
what it was designed to be: a **distributed, federated network** —
form factors are separate products, but the Mesh is one federated
whole. The unit of federation is the **fedinet**: a named set of
nodes whose membership is kept by one node's **gateway** as a
**signed roster**.

## The roster is the gateway's signed word

A roster carries, for every member: its node id, its name, its
public key and door-certificate fingerprint, its addresses and
declared public endpoint, its capabilities, when it joined — and
its **Node number**, the join order the gateway assigned at
admission. Node numbers are *roster facts*: they exist only
inside the signed roster, nowhere else.

The gateway signs every roster revision with its node key.
Versions only move forward. A member accepts a roster only
whole: right fedinet, the gateway's signature verifying,
version strictly newer than the one it holds. A forged,
unsigned, or rolled-back roster is refused by name at the door
and the stored roster stands.

Rosters travel over the Mesh door (`POST /mesh/roster`), signed
like every Mesh request. Authentication at the roster door is
deliberately wider than pairing: a fedinet member is known by
the roster itself — its key is a roster fact — so a member this
node has not paired with yet can still sync, knock, and be
answered. Asks go **direct first**, then **via relay**: Build 8
admits the roster class to Build 7's sealed relay carriage, on
the same three consent ticks and the same one-hop rule, with
the destination checking its own relay consent at its roster
door. Rosters also gossip member-to-member (the `roster`
action), so a fedinet heals around a dark gateway; the gateway's
signature is the only credential an offered roster needs.

## Gateway cards — admission + introduction, never trust

A gateway hands out a **gateway card**: a pairing card's form
rendered **green**, the fedinet's name on its face, and a
`navi-gateway-card` payload inside the picture — the gateway's
own pairing card, the fedinet's identity, and the gateway's
signature over the claim.

The card's reachability is the gateway's **declared public
endpoint alone** (v0.4.4e, #238). A pairing card carries the
machine's LAN addresses because pairing is arranged between
machines whose operators can meet on a network; a gateway
card is imported by machines *outside* the gateway's network,
and a member's bridge dials the card's addresses in order —
a LAN address on it is a door that exists in only one house.
On the member's own LAN such an address is a connection
sitting in SYN-SENT until it times out, or a stranger's
machine holding the same private address, and the automatic
route can fall back to a local answer without the public
road ever being tried. So the local addresses come out of
the gateway's card entirely — the embedded pairing card
carries the endpoint as its only address, and the gateway's
own roster entry is shaped the same way. A gateway with **no
declared public endpoint** mints no gateway card at all: the
mint refuses, on the Federation page and at the download
alike, with a sentence naming the missing declaration.
Declare the endpoint on the Mesh page and the card can be
minted. (Pairing cards are unchanged: they keep their LAN
addresses by design.)

Importing a gateway card does exactly two things: **admission**
(this node now keeps the fedinet) and **introduction** (the
gateway becomes a peer, and its roster becomes fetchable). It
grants **no trust** — no lending posture moves at import.
Trust is a separate act, below.

A gateway that founded its fedinet is also its **Founding
Member** — two distinct facts, both visible: the roster lists
the gateway as Node 1 with the Founding Member mark, and the
Federation page says both.

## Joining: the knock, and the gateway's word

A node admitted by a gateway card **knocks**: it sends its own
pairing card, self-signed, to the gateway's roster door. The
knock proves the knocker holds the key its card names — and
nothing more; admission is the **gateway operator's act**, on
the Federation page's pending list (admit or deny). Admission
adds the member to the roster at the next Node number, pairs it
on the gateway, and republishes the signed roster.

**When the road is dark (v0.3.0e — Build 6, #226).** A knock or
a roster sync that cannot reach the gateway names the stage it
died at and the last road tried — nothing answered in time at
an address, no connection to be had, a TLS handshake that
failed, a door that answered without the pinned certificate, a
connection dropped mid-answer — instead of every failure
reading the same *did not answer*. A failed knock is written
to the node's federation audit log with its stage, one entry
per operator-initiated attempt, the same cadence a failed sync
already kept: silence in the log once again means no knock was
sent.

**Cards keep their kinds, and verified cards refresh
(v0.3.0e — Build 6, #227).** A gateway card can never become a
hand-made pairing record, and a pairing record can never serve
as a fedinet's gateway record: every import path — the pairing
picture import, the federation import, backup restore, and
sync-carried cards — refuses the other kind by name. Gateway
cards are imported on the Federation page, where the gateway's
signature is checked; pairing cards pair one machine from the
Mesh page. And when a *verified* gateway card's node already
has a peer record on the importing node, the import refreshes
that record's card-claimed reachability — addresses, port,
certificate fingerprint, name — from the card, because the
knock dials the record, never the card: a record made before
the federation existed (a hand pairing carrying only the
gateway's LAN address) otherwise keeps its stale roads
forever. Operator facts are never the card's to change: relay
ticks, the tunnel route, borrow posture, pair history, and
the auto-paired mark all stand. The Mesh health check names a
federation-marked record whose stored addresses are a proper
subset of its verified card's, and offers the same refresh as
an in-place heal on that peer's card.

**Expulsion** is a signed roster revision like any other: the
member leaves the roster, and at every member's next sync the
federation-made records for it go away. Pairings made by hand
outside the fedinet are never touched — one that was later
admitted is *unmarked*, not deleted.

## Deep trust — and its guardrails

Trust-by-default is **deep** (the founder's ruling, 2026-11-13,
taken with its threat model on the table): when an operator's
toggle for a fedinet is ON, that fedinet's signed roster
**auto-pairs and auto-lends** on their node. The federation
materializer reconciles the node's peer records with the
roster — it only ever touches records the federation itself
made. Lending is re-judged fresh on every borrow ask against
this node's own posture:

- **The roster's freshness.** A roster last verified more than
  **72 hours** ago degrades the fedinet to pairing-only, with
  the sentence on the Federation page saying so. Pairings hold;
  auto-lending waits for a sync.
- **The new-member tick.** Each operator's per-fedinet tick —
  "lend to new members automatically" — defaults ON. A member
  whose first-seen roster version is newer than the operator's
  trust baseline is *new*: unticked, new members are
  pairing-only *for that operator* until ticked individually.
- **Per-member untrust** always wins — over the toggle, the
  tick, everything.
- **A per-member lend tick** names an exception: that member
  lends even while the general toggle is off. It is not
  auto-lending; it is named lending, and untrust still beats it.
- **Borrow caps** are local policy: a per-fedinet hourly cap on
  generations lent to its members, counted from the ledger.
- **Owner preemption.** The owner's own work goes first: local
  generation starting aborts a federation borrow in flight,
  and the borrower is answered with the named outcome
  *preempted* — never dropped silently. A hand-made pairing's
  borrow is never preempted.

One flip of trust-by-default OFF ends all auto-lending for the
fedinet. Pairings already made stay pairings.

## Founding, and Rockport Mesh

Anyone can found a fedinet: name it, and your node becomes its
gateway, signs roster v1, and stands as its Founding Member.
A node keeps several fedinets at once; no trust bleeds across
them — every posture above is per-fedinet, per operator.

**Rockport Mesh** is the first fedinet. The product ships the
machinery and Rockport's *definition* — never a hardcoded
roster: its real roster lives on its gateway (Neo-Arcadia) and
is learned from the gateway's signature. First runs meet
Rockport through its gateway card.

## The Federation page, and the doors

Federation is administered on its **own page** (Apex |
Systems → Federation) — fedinets and rosters, gateway cards,
toggles and ticks, founding controls, caps, pending knocks, and
the **trust audit log**. The phone keeps a dedicated
arrangement of the same cards. Since DHD serves the same UI,
the page is also the remote admin surface — for the operator's
**own nodes only**: every act reads and writes the node it
lands on, so no door can flip another operator's switches.

The tiers (the crown-acts ruling, 2026-11-13):

- **Read + revoke** are open through DHD always.
- **Manage** acts — imports, ticks, caps, admissions — need a
  trusted-device session **plus the Operator Badge step-up**.
- **Crown acts** — enabling deep trust, founding a fedinet,
  authorizing a tunnel key — run through DHD under the full
  ceremony: trusted-device session + Badge presentation + a
  **named-action confirm** (the act spelled in words) + an
  audit entry. At the desk, the same acts need the same
  confirm + audit; the Badge is the door's addition.

The Badge factor is the operator's **Operator Badge** (the
operator card). The node stores only its **SHA-256
fingerprint**, enrolled **in person** on the Federation page —
enrollment itself never rides DHD. A presented Badge is hashed
and compared; the picture never stays on the node. With no
Badge enrolled, DHD manage and crown acts refuse **by name**:
the ceremony cannot run, and says so.

Every trust-class act, through any door, lands in the audit log
(`config/federation/audit.json`, the last 300 rows) — action,
fedinet, detail, and which door, in sentences. Metadata only.

## The Docker invitation

An image built to join a fedinet carries the gateway's card at
`docker/gateway-card.png`. The container's first boot copies it
into the volume as the waiting invitation
(`config/federation/invitation.png`), and the bridge imports it
at start — admission-only. Under the container's ruled Rockport
default, the imported fedinet's trust-by-default starts ON; it
engages only when a signed roster syncs, and the import itself
lends nothing. Node keypairs still mint at first boot and are
never baked into any image; the gateway's identity never leaves
the gateway — its card is minted in the field, on the gateway
itself. The tree's `docker/sample-gateway-card.png` is a
**sample** of the card's shape carrying sample facts, clearly
named as such, and is never wired as an invitation.

## "Mesh Online"

While a node keeps any fedinet, the Mesh status headline is
the state — **Mesh Online** — not a census of pairs. Warnings
still outrank it. A node keeping no fedinet speaks exactly as
it did before federation.
