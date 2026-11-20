# app/index.html — part 5 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 7306–9258 of app/index.html.

  "<p><b>The operator class converges</b> (v0.5.1e, #256). What she knows about <i>you</i> — your name, your picture, your badge, and her memory entries about you — is one class of its own, and since this build it actually crosses. The bridge's seat record is the truth: a window's list is a cache of it, written through on every change and replaced by the converged answer, so a fact added at the server is simply <i>there</i> at the laptop — no seat keeps a private list the sync merges underneath. The status pane's <b>MEMORY</b> line reports the class's true state — the entry count and the sync clock, in the sync bar's own terms — and the <i>(local)</i> it used to confess to is retired. On the backup road the same discipline names its keys: an import applies a deliberate list — your name, threat level, sudo mode, installed commands, updater channel — and nothing else in the config crosses; a machine keeps its own hardware profile, its Mesh identity, and its install facts, always.</p>" +
  "<p><b>Fluid sync — push-on-write</b> (v0.5.1e, #257). The occasions above still fire, but a change no longer waits for one: a <b>write</b> on any seat — a chat message, a memory entry added or forgotten, a case change, a deletion — asks for a round of its own, promptly, and the receiving seat converges <b>inside ten seconds</b> while both seats are up and in Mesh contact. The Mesh is the primary road; the repo push is the writing seat's fallback act, and a seat off both roads converges at its next occasion, as before. The standard is the account's, not one operator's: it holds for anyone with an account, on their own seats. Every landing receipt on the sync bar names the road the round took and the time it measured — <i>landed in 3s via Mesh</i> — so the standard is checkable at a glance, and a push that cannot land names its stall on the bar rather than silently aging. No daemon was added: the push rides the bridge's own resident life, and a burst of writes coalesces into one round.</p>" +
  "<p><b>Substance, not shells</b> (v0.5.13e, #262). A round used to converge whatever arrived and call it synced — and once, a case chat's <i>existence</i> crossed to the phone while its transcript stayed behind, and a theme never crossed at all. Now the set's manifest <b>declares the substance</b>: every chat's message count and a digest of its messages, and a digest of the seat (your settings and theme) and of the Pulse store. The receiving side <b>verifies arrivals against the declaration</b> — a chat whose count or digest falls short is named <b>incomplete</b> on the sync bar and in the record; a seat or Pulse payload whose bytes fail their digest is refused rather than merged over good local substance; and where this seat already holds more of a conversation, a short copy never replaces it. A shell is not a dead end: the next round's <b>full arrival heals it in place</b>, and the name clears. Sets from older builds declare nothing and are taken as they come, as before.</p>" +
  "<p><b>The override — Send or Receive</b> (v0.6.2e, #303). Sometimes you do not want the cadence's timing; you want it <i>now</i>, in one direction. A setting — <b>Settings → GitHub &amp; Backup → The Sync button</b>, on by default — makes the <b>Sync</b> control offer a choice when pressed: <b>Send a sync</b> publishes this seat's set and pulls nothing down; <b>Receive a sync</b> pulls the converged set and publishes nothing. Each is the standing machinery standing alone by direction, over the same roads, and each lands with its own receipt naming what actually moved — files published, what came down — or saying plainly that there was nothing to move. Turn the setting off and a press runs the whole round at once, as it always did. The automatic rounds never consult the setting.</p>" +
  "<p><b>The override overrides</b> (v0.6.3e, #317). A chosen direction does not queue behind a round already in flight: the running round stops at its next phase boundary — its partial work stands, since convergence is idempotent — and your Send or Receive runs immediately after. Both receipts tell the truth: the stopped round's record names the supersession and the phase it stopped at, and its card face reads <i>Overridden — your call runs now</i>, never <i>held</i>. Choose again mid-flight and the newest choice is the one that runs. Machine triggers — push-on-write, the occasions — still hold behind a running round, as they should: preemption is the operator's privilege, exercised through the choice.</p>" +
  "<p><b>A cleared main chat stays cleared</b> (v0.6.3e, #318). Emptying the main thread with <span class='mono'>/clear</span> or <span class='mono'>/cls</span> is a synced act: the thread carries a clear marker — the moment it was emptied, and the seat that emptied it — and convergence honors it on every seat. Content stamped at or before the clear yields to the emptied state however new the copy carrying it claims to be; lines written after the clear are untouched, and the ghost-fold can never resurrect what a clear retired. And the verdict is honest about it: after every round the main thread's count and digest are checked against the converged set, and on a disagreement the round is renamed — <i>Mostly synced — the main thread disagrees</i>, with both counts named — on the receipt and on the Sync card. A clean <i>converged</i> is only ever printed over a main thread that actually agrees.</p></div>" +
    "<div class='doc-sec'><h4>One sign-in, and failures with names (Build 2, completed in Build 6)</h4>" +
  "<p><b>Who Apex speaks as.</b> Build 2 taught Apex to keep its GitHub identities apart — your sign-in for backup and import, a separate pasted token for the app's own reads — and Build 6 finished the thought by collapsing them to <b>one</b>: your GitHub account sign-in, and nothing else. There is no token to create, paste, or rotate; the Updater's checks, the NUR's catalog searches, downloads, sync's repo transport, backup and import all ride the one sign-in, which lives in <span class='mono'>config/github-auth.json</span> (readable only by you, never backed up, deleted by signing out). <b>Release Automation</b> still holds no credential in Apex at all, by design: releases are published from your signed-in browser session and the local git remote. And failures keep their names everywhere GitHub is read: <b>auth</b> (the credential itself is refused), <b>rate</b> (a bucket is spent — the sentence carries its reset), <b>abuse</b> (GitHub's secondary safeguard — the sentence carries the wait it asked for, and Apex stops rather than retrying). No surface shows a bare “HTTP 403”.</p></div>" },
  { id: "server-mode", title: "Server mode — headless & at boot", related: ["installing", "dhd"], html:
    "<div class='doc-sec'><h4>Apex without the window</h4>" +
  "<p>A server does not need a window. <span class='mono'>./start.sh --headless</span> starts exactly the services — the llamafile (check-first, as always: an answering server is left alone) and the bridge, supervised — and opens <b>no window</b>. It is safe over SSH, safe to run twice (a service already answering is adopted, never doubled), and it writes the same <span class='mono'>start.log</span> the desktop launch writes, so a headless start explains itself after the fact the same way. Run it after a reboot and Apex — with DHD, if DHD was left on — is simply back.</p>" +
  "<p>The desktop launch adopts, too: open the app on a machine whose services are already running (from boot, or from a headless start) and it uses them as they are instead of starting second copies.</p></div>" +
    "<div class='doc-sec'><h4>At boot, honestly</h4>" +
  "<p>Choose <b>At system boot</b> in the installer and it registers <b>two</b> systemd user services — <span class='mono'>navi-llamafile.service</span> and <span class='mono'>navi-bridge.service</span> (the bridge unit runs <span class='mono'>start.sh --headless</span>) — and tries to enable <b>lingering</b> for your user, which is what lets user services start at boot rather than at login. If the installer cannot enable lingering itself, it prints the exact command — <span class='mono'>sudo loginctl enable-linger $USER</span> — rather than assuming: a boot-armed Apex you only <i>believe</i> in is worse than one that tells you the step it could not take. The status column's <b>Servers</b> group carries a <b>Boot</b> row that reads the armed state from the system every time — both units and the linger state — so what it shows is what the machine says, never what an installer once hoped.</p><p>The app can now change it. <b>Arm at boot</b> and <b>Disarm</b> in the status column's Servers \u2192 Boot row \u2014 and the same control in Settings \u2192 System \u2014 register and enable both units from the templates the package ships, and try to enable linger. If linger cannot be enabled from the app, the row shows the exact command \u2014 <span class='mono'>sudo loginctl enable-linger $USER</span> \u2014 instead of pretending. After any action the row re-reads the system, and a half-done arm shows as half-done. Disarm disables the units \u2014 what is running keeps running; it just won't come up at boot \u2014 and leaves linger as it is. Arming is a local action \u2014 it is not offered through the DHD door.</p></div>" },
  { id: "headless-autostart", title: "Headless auto-start — setup, the Wi-Fi secret fix, and the boot test", related: ["server-mode", "dhd", "troubleshooting"], html:
    "<div class='doc-sec'><h4>Headless auto-start</h4>" +
    "<p>Apex's headless mode starts the bridge on boot, with no app window and nobody logged in — so a machine can power on and simply <i>be on duty</i>: answering locally, over DHD, and lending to the Mesh. This page covers the one-time setup, the one environment trap that can silently defeat it (a Wi-Fi profile with no stored secret), and the boot test that proves the whole chain. Everything below is a one-liner where a command is needed.</p></div>" +
    "<div class='doc-sec'><h4>1. One-time setup</h4>" +
    "<p>Boot-arming lives <b>in the app</b>. In the status column's <b>Servers</b> group, the <b>Boot</b> row carries an <b>Arm at boot</b> button \u2014 the same control sits in <b>Settings \u2192 System</b>, on the <b>Startup \u2014 at boot</b> card. Press it and the app:</p><ul>" +
    "<li>registers and enables two systemd user services \u2014 <span class='mono'>navi-llamafile.service</span> and <span class='mono'>navi-bridge.service</span> (the bridge unit runs <span class='mono'>start.sh --headless</span>) \u2014 from the templates the package ships, and</li>" +
    "<li>tries to enable <b>lingering</b> for your user, which is what lets user services start at boot rather than at login. If linger cannot be enabled from the app, the row shows the exact command instead of pretending:</li></ul>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>sudo loginctl enable-linger $USER</pre>" +
    "<p>After any action the row re-reads the system, so what it shows is what the machine reports now \u2014 and a half-done arm shows as half-done. <b>Disarm \u2014 don't start at boot</b> disables the units (what is running keeps running; it just won't come up at boot) and leaves linger as it is. Arming is a local action \u2014 it is not offered through the DHD door. The installer remains the alternative path: run it and choose <b>At system boot</b>, and it registers the same units and tries linger the same way, as the initial state.</p>" +
    "<p>You can also start the services headlessly at any time by hand — safe over SSH, and safe to run twice (a service already answering is adopted, never doubled):</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>./start.sh --headless</pre>" +
    "<p><b>Check the armed state in the app, no terminal needed:</b> the status column's <b>Servers</b> group carries a <b>Boot</b> row that reads the state from the system every time it shows — <i>Armed — Apex comes up headless at boot</i>, or <i>Not armed — Apex starts when you open it</i> — with a detail line naming both units and the linger state (on/off), read live, never assumed.</p>" +
    "<p><b>Reaching the machine from your own network:</b> if you want other devices (a phone, a second laptop) to reach this machine without DHD, turn the <b>Local server</b> toggle on in <b>Settings → System</b>. Off (the default), the bridge listens only on the machine itself (<span class='mono'>127.0.0.1:8675</span>); on, it answers on the LAN on port <b>8675</b>, and anything that isn't the machine itself must present your Apex access code first.</p>" +
    "<p>Verify the boot machinery from a terminal:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>systemctl --user is-enabled navi-bridge.service navi-llamafile.service\nloginctl show-user \"$(id -un)\" --property=Linger --value</pre>" +
    "<p>You want <span class='mono'>enabled</span> twice and <span class='mono'>yes</span>. If the linger query gives you trouble, the ground truth is the directory — your username listed means linger is on:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>ls /var/lib/systemd/linger/</pre>" +
    "<p>If the Boot row says <b>Not armed</b>, press <b>Arm at boot</b> on the row (or in Settings \u2192 System) \u2014 re-running the installer and choosing <b>At system boot</b> is the fallback if the units cannot be registered from the app. If it is armed but linger is off, run the <span class='mono'>enable-linger</span> command the row shows, once.</p></div>" +
    "<div class='doc-sec'><h4>2. The trap: a Wi-Fi profile with no stored secret</h4>" +
    "<p>Headless Apex can only answer on a network that exists. On Pop!_OS / Ubuntu (the netplan-backed NetworkManager stack), a Wi-Fi connection created without a system-stored password keeps its secret in the <i>desktop session</i>. At boot, NetworkManager asks for the secret, no session agent exists yet, and activation fails with:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>&lt;warn&gt;  no secrets: No agents were available for this request\n&lt;info&gt;  Activation: failed for connection '&lt;SSID&gt;'</pre>" +
    "<p>NetworkManager then backs off and retries much later — so after a reboot the machine is dark to SSH, DHD, and the Mesh until a human logs into the desktop, at which point everything \"mysteriously\" comes back at once. This is an environment defect, not an Apex defect: the bridge started fine; the network underneath it never arrived.</p>" +
    "<p><b>Check whether your profile stores its secret:</b></p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>sudo grep -c password /etc/netplan/90-NM-*.yaml</pre>" +
    "<ul><li><span class='mono'>1</span> or more — the password is stored in the system profile. You're clean.</li>" +
    "<li><span class='mono'>0</span> — the profile is secretless. Apply the fix below.</li></ul>" +
    "<p>(The netplan source file is named <span class='mono'>90-NM-&lt;connection-uuid&gt;.yaml</span>. Wired connections don't have this problem; it is Wi-Fi-specific.)</p>" +
    "<p><b>The fix.</b> Type this yourself, at the machine, with your real Wi-Fi password — never send the password through a chat, a ticket, or a screenshot:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>sudo nmcli connection modify \"&lt;SSID&gt;\" wifi-sec.psk \"&lt;your-wifi-password&gt;\"</pre>" +
    "<p>On the netplan-backed stack this writes through to the netplan source file, so the generated boot profile carries the secret and NetworkManager can activate Wi-Fi with nobody logged in. Expect the connection to bounce once when you apply it (an SSH session may drop; reconnect).</p>" +
    "<p>Verify the write landed — again without printing the secret: run the <span class='mono'>grep -c</span> check above again. You want <span class='mono'>1</span> or more now.</p></div>" +
    "<div class='doc-sec'><h4>3. The verdict test (local — no DHD required)</h4>" +
    "<p>This proves auto-start on one machine using only a phone:</p><ol>" +
    "<li>Reboot the machine.</li>" +
    "<li><b>Do not log in.</b> Leave the login screen completely untouched — hands off is the whole test. If you log in, you've proven nothing.</li>" +
    "<li>Wait about two minutes.</li>" +
    "<li>On your phone, on the <i>same Wi-Fi</i>, open <span class='mono'>http://&lt;machine-LAN-IP&gt;:8675</span> and enter your Apex access code. (Requires the <b>Local server</b> toggle from §1.)</li>" +
    "<li><b>Pass:</b> Navi answers while the login screen still sits untouched. The node auto-fired: service up, network up, no human involved.</li></ol>" +
    "<p>Wrinkle: some networks (many workplace and guest Wi-Fi setups) isolate clients from each other, so the phone can't see the machine. Then put the machine on your phone's hotspot instead and browse to it there — same test, your own network.</p>" +
    "<p><b>The remote variant.</b> Once the local test passes, the same proof works from anywhere, through whichever doors you've set up — from another machine, try your SSH door and/or your DHD address after a no-login reboot:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>ssh -p &lt;your-ssh-port&gt; &lt;user&gt;@&lt;your-host&gt;</pre>" +
    "<p>If either answers with the login screen untouched, headless auto-start is proven end to end on that machine.</p></div>" +
    "<div class='doc-sec'><h4>4. If it fails</h4>" +
    "<p>Log into the desktop — everything recovers; that recovery is itself a clue (it means the services were waiting on something the login provided). Then read the boot record before changing anything:</p>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>journalctl -b -u NetworkManager --no-pager | grep -i -E \"no secrets|activation.*fail\"\nsystemctl --user status navi-bridge --no-pager</pre>" +
    "<ul><li>The <span class='mono'>no secrets</span> line is present → §2's trap; apply the fix and retest.</li>" +
    "<li>The bridge unit is failed or missing → the problem is in the Apex layer; the unit's own journal lines say why.</li>" +
    "<li>Neither → note the exact boot time and what <i>did</i> answer, and file it with those two outputs attached. Guessing wastes more time than the journal takes to read.</li></ul></div>" +
    "<div class='doc-sec'><h4>5. One-liner summary</h4>" +
    "<ul><li>Boot units armed? — <span class='mono'>systemctl --user is-enabled navi-bridge.service navi-llamafile.service</span></li>" +
    "<li>Linger on? — <span class='mono'>loginctl show-user \"$(id -un)\" --property=Linger --value</span> (fallback: <span class='mono'>ls /var/lib/systemd/linger/</span>)</li>" +
    "<li>Turn linger on — <span class='mono'>sudo loginctl enable-linger $USER</span></li>" +
    "<li>Wi-Fi secret stored? — <span class='mono'>sudo grep -c password /etc/netplan/90-NM-*.yaml</span></li>" +
    "<li>Store the Wi-Fi secret — <span class='mono'>sudo nmcli connection modify \"&lt;SSID&gt;\" wifi-sec.psk \"&lt;your-wifi-password&gt;\"</span></li>" +
    "<li>Bridge alive right now? — <span class='mono'>systemctl --user status navi-bridge --no-pager</span></li>" +
    "<li>Boot-time network failures — <span class='mono'>journalctl -b -u NetworkManager --no-pager | grep -i -E \"no secrets|activation.*fail\"</span></li></ul></div>"
  },
  { id: "installing", title: "Installing & updating", related: ["getting-started", "troubleshooting"], html:
    "<div class='doc-sec'><h4>Installing Navi, and keeping her up to date</h4>" +
  "<p>Navi lives on your own computer — the app, her model, her memory, and her voice are all on your machine.</p>" +
  "<p><b>Right now Navi installs on Linux.</b> That is the one installer in the package: <span class='mono'>install.sh</span>. How you run it depends on where your copy came from — from the <b>release zip</b>, run <span class='mono'>./install.sh</span> (its scripts arrive executable, so that just works); from a <b>git clone or GitHub's source zip</b>, run <span class='mono'>bash install.sh</span> instead (the paragraph below explains why, in plain words). Windows and macOS installers are planned for later — they are not in this package yet, so there is nothing to run on those systems today.</p><ol>" +
  "<li>Download the app package and unzip it — or clone the repo / unzip GitHub's source zip, and use <span class='mono'>bash install.sh</span> in the next step instead of <span class='mono'>./install.sh</span>.</li>" +
  "<li>Run the Linux installer: <span class='mono'>./install.sh</span> from the release zip, or <span class='mono'>bash install.sh</span> from a clone or source zip. It asks one question: should her model start when the app opens, or when the computer boots? Either is fine.</li>" +
  "<li>The installer downloads her model file (big — about 3 GB — so this takes a while) unless it is already there. It never downloads it twice.</li>" +
  "<li>It also offers her voice, and sets up the tool that builds her window, the same way — say no and she still works; she just cannot talk out loud yet.</li>" +
  "<li>It looks at your machine — the processor and its cores, how much memory, the graphics chip — and picks her thinking settings to fit it: how many threads, how big a context, how much she reads at once, and whether the graphics chip helps. A modest machine gets the gentle, proven numbers; a strong one gets more. It writes what it found and why into <span class='mono'>hardware-profile.log</span>, and you can see or change the result later in Settings.</li>" +
  "<li>Open <b>Apex</b> from your app menu. Her own window opens — no web browser. That window is the app.</li></ol>" +
  "<p>On Linux she installs into <span class='mono'>~/.local/share/navi-apex</span>, your own data home. Her window talks to a small helper program (the <b>bridge</b>) at port " + BRIDGE_PORT + ", and the bridge talks to her model at port 8080. <b>Your things are safe:</b> updating, reinstalling, and even a fresh install (<span class='mono'>--fresh</span>) keep what is yours — her model file, installed chips and knowledge packs, the Vault, the DHD certificate home (<span class='mono'>config/dhd/</span> — a real certificate there is never wiped by a reinstall), her Soul and Memory pages, her voice and its models. A first install is the only true blank start.</p>" +
  "<p><b>Permissions are audited, not discovered.</b> Before an update or a migration stops or carries anything, the installer audits the whole source tree in one pass for files you cannot read — the leftovers of a past <span class='mono'>sudo</span> repair are the usual suspects — and handles each by its class. <b>Bytecode</b> (<span class='mono'>__pycache__</span>, <span class='mono'>.pyc</span>) is simply never carried; it rebuilds itself. A <b>re-issuable credential</b> she cannot read — the GitHub sign-in file — is left behind, and the install's completion output and record name the casualty and the fix (sign in again from Identities). <b>Irreplaceable state</b> she cannot read — the Mesh identity, cards, chats, Case Files, Soul and Memory, the rest of config — refuses the install up front, naming <i>every</i> offending file with the one <span class='mono'>sudo chown</span> command that repairs them all: never one file per attempt. After the install, her self-check adds an <b>ownership verdict</b> over the new tree, so a file that somehow lands root-owned is named by the product itself. An unprivileged installer cannot repair another user's ownership — the doctrine is that it never needs to, except for irreplaceable state, and there the stop comes first, complete, with its remedy attached.</p>" +
  "<p><b>One installer, every major Linux family.</b> The installer reads your distro from <span class='mono'>/etc/os-release</span> and says out loud which family it found. Only its dependency step changes with the family — the same development libraries and base tools, under each family's own package names:</p><ul>" +
  "<li><b>Debian family</b> (Debian, Ubuntu, Pop!_OS, Mint and their kin) — installed with <span class='mono'>apt</span>.</li>" +
  "<li><b>Fedora</b> and Fedora-based distros — installed with <span class='mono'>dnf</span>.</li>" +
  "<li><b>Arch</b> and Arch-based distros — installed with <span class='mono'>pacman</span>.</li>" +
  "<li><b>Bazzite</b> — Fedora-based but immutable, so it is its own case (below).</li></ul>" +
  "<p>Everything else — profiling your machine, the model download, what an update keeps, the startup choice — is the same on every family. On <b>Bazzite</b> the installer never layers packages onto the system with <span class='mono'>rpm-ostree</span> and never asks for a reboot: her window is built inside a Fedora <b>distrobox</b> when you have one, and when you don't, the window build is skipped — in plain words, during the install — and you get the complete core install: the bridge and the dashboard, which <span class='mono'>start.sh</span> opens in your browser. Create a Fedora distrobox later and re-run the installer to add the window. And if your distro isn't one the installer recognizes, it says so, guesses no package manager, installs everything it can user-space, and completes the core install just the same.</p>" +
  "<p><b>Installing from a git clone — or from GitHub's source zip</b> (the zip the green Code button offers, the one that unpacks to a folder like <span class='mono'>Apex-main</span>) <b>works too</b> — with two honest differences. <b>First, how you run it:</b> from a clone or a source zip, the way is <span class='mono'>bash install.sh</span> — and <span class='mono'>bash start.sh</span> when you start her from that source tree. Run it through bash because it needs no executable permission. Why, in plain words: the git tree does not preserve the scripts' executable permission, so every script that comes out of a clone or a source zip arrives without it, and a script without that permission cannot be run as <span class='mono'>./install.sh</span> until the permission is set — running it through bash skips that entirely, because bash reads the file itself. From the <b>release zip</b> it is the other way round: its scripts arrive executable, so <span class='mono'>./install.sh</span> is the instruction there. Release zip → <span class='mono'>./install.sh</span>; clone or GitHub source zip → <span class='mono'>bash install.sh</span>. If you try <span class='mono'>./install.sh</span> from a clone anyway, you will meet a baffling pair of errors — bash answers <i>Permission denied</i>, and <span class='mono'>sudo ./install.sh</span> answers <i>command not found</i>, which sounds like the file is not there at all. It is there; it just cannot be run that way. Use <span class='mono'>bash install.sh</span> and both errors go away. (If you prefer <span class='mono'>./install.sh</span>, you can set the permission yourself first — <span class='mono'>chmod +x install.sh start.sh run-llamafile.sh</span> — and then run it the usual way. That works too; bash is simply the shorter path, and the one these instructions lead with.) <b>Second, the icons:</b> the clone does not carry the binary icon files (the app icon, its <span class='mono'>.ico</span>/<span class='mono'>.icns</span> renditions, and her window's icon): those ship only inside the release zip. So a clone install runs without the custom icon — the installer says so in plain words while it installs, the app and its menu entry use a fallback icon, and her standalone window is not built, because it is built around that icon; she opens in your browser instead. Everything else about the install is exactly the same. The release zip remains the complete package.</p>" +
  "<p><b>Updating is real, and it works like this.</b> The <b>Updater</b> tab (and one quiet automatic check each time she starts) asks GitHub for the project's latest release and compares its tag with her VERSION file. If a newer release exists, the single line under Navi's status in the status column flips to a yellow <b>Update available</b>. The Updater screen shows that release's <b>notes</b> right there — the words from its GitHub release page, or, when the release page carries none, the matching section of the project's CHANGELOG — so you can read what changed before you press the button. Press <b>Download &amp; install</b> and she does the whole job from that one click: the release package is downloaded, checked (it must be a real Apex package — and a package that arrives gzip-wrapped is unwrapped and accepted, not refused), and unpacked beside the app, under <span class='mono'>updates/</span>; then a small helper that outlives the app runs that folder's installer — never with <span class='mono'>--fresh</span>, so everything that's yours is kept, exactly as the paragraph above promises — and relaunches her. While it works, the Updater names the step it is on: downloading, verifying, staging, installing, restarting. When she comes back, the Updater shows the confirmation in plain words — <b>Updated v0.1.6a → v0.1.7</b>, with the actual versions — and only when the install after the restart reads as what the update promised: the VERSION file's version — and, for a same-version build refresh, the build number the relaunched bridge reports and the one baked into the bridge on disk, since a refresh deliberately keeps the same version number. Before staging, the package itself is proven to be the build on offer: a package left beside the app by an earlier update is reused only when its size matches the offered package's, and the build number inside its bridge must match the release's, or it is fetched fresh. If any step fails, the Updater names the step and the reason, nothing is left half-applied that can be rolled back, and — once the package was staged — it shows the manual fallback as a fallback: the staged folder and the one command that finishes the job by hand, running that folder's installer with <span class='mono'>bash install.sh</span> in a terminal. That command carries your startup and voice choices as flags (<span class='mono'>--startup</span> / <span class='mono'>--voice</span>), so the installer never has to ask a question — which is the whole point in a shell with no terminal to answer one, like her own CLI shell: there, an installer that asks dies at the first question (issue #75, field-proven on a staged update), while the flagged command runs straight through. The helper keeps a <b>step log</b> while it works — <span class='mono'>apply-trace.log</span> in the update's folder under <span class='mono'>updates/</span>, its path named on the apply record — timestamping every step it completes, with a heartbeat while it waits, so an interrupted apply shows exactly where it stopped; if the helper itself fails or is stopped, the record is closed as failed at that step with the reason in words, and if the helper is killed outright — the one death it cannot record itself — the bridge's <b>apply watchdog</b> closes the record the same way, quoting the step log's last line, instead of leaving it claiming to be installing forever. A relaunched bridge that never answers fails the record at the restarting step, naming the build that was expected and what was found instead. And if a check cannot reach GitHub — no network, or GitHub's rate limit — the Updater says that, in those words, instead of pretending you are up to date.</p>" +
  "<p><b>An apply can no longer kill itself (Build 12, #201).</b> The helper descends from the app it replaces, and the installer descends from the helper — so an install step that stops the app's <i>service</i> (which signals every process in the service, as a group) kills the whole family mid-flight: exactly that ended the first homecoming pull on both of the Row's machines, two seconds in, before the migration carried a single byte. The rule now is stated in the code and kept in both halves of the apply: an apply may end exactly one process by signal — the bridge it replaces, by its proven pid — and never a group-shaped signal: not a unit stop, not a process-group or cgroup kill. In practice: the bridge spawns the helper <b>outside its own systemd unit</b>, in a transient scope of its own (via <span class='mono'>systemd-run</span>), so stopping the app's unit during an install cannot reach it at all; where no scope can be had (no user manager answers — the container shape), the spawn says so in the step log and the installer's ordering carries the run instead: the installer stops the old bridge <b>by pid</b> first, and only then retires the old boot services' files — <span class='mono'>disable</span> and removal, never a stop — re-proving afterwards that nothing resurrected. The same round closes the homecoming's last soft spot: a new home left standing <b>without its migration record</b> is an <i>interrupted</i> home, never a canonical one — a later install runs the migration again and carries state from the old tree, rather than converging onto a half-copied tree and calling it home.</p>" +
  "<p><b>The incoming build governs its own installation (v0.3.0e Build 1, #205).</b> Build 12's law has a limit the field named: a pull is executed end-to-end by the machinery of the build that is <i>running</i> — its spawner, its helper — so a spawner fix cannot protect the pull that installs it; on one of the Row's machines, Build 12's fixed spawner never got to run its own arrival. From this build, the package carries its apply machinery with it: <span class='mono'>apply-bootstrap.py</span> and <span class='mono'>apply-helper.py</span> beside the payload. When a bridge stages a package that carries them, the package's own bootstrap spawns the package's own helper — same spawn discipline, the package's bytes. And when an <i>older</i> build's helper runs this package's installer, the installer's first act — before anything is stopped — is the <b>bootstrap handoff</b>: the bootstrap spawns the new helper in takeover mode and the old installer's process exits, having installed nothing. The new helper never signals the old one; it waits for it to finish its own bookkeeping and die — the no-ancestry law is untouched, because a predecessor is not an ancestor — and then runs the real install itself, from outside the old bridge's unit. If a handoff cannot arm, the ordinary install carries on: a handoff can never strand an apply. The apply record names how the helper was spawned — <span class='mono'>package-bootstrap/…</span> when the incoming machinery governed, the plain Build 12 modes when it did not — and the step log tells the whole story, takeover lines included.</p>" +
  "<p>Since Build 3, the failure card pairs its fix with the reading: beside the manual-fallback command it shows a labeled <b>See what happened:</b> command — one copy-paste line that prints the failed apply's own logs (the installer's output and the helper's step log, from the staged update's folder). Read first, then run the fix with the whole story in view. Build 3 also closes the field failure behind that card (issue #174): the installer now stops the running window with the same care as the bridge before replacing files, and installs the window's binary by write-to-temp and atomic rename, so a still-running window can never again make the copy fail with <i>Text file busy</i>; and when an install runs detached — no terminal, as the Updater's helper does — the log says plainly that a detached apply cannot answer a sudo prompt when the system-package step is skipped for exactly that reason.</p>" +
  "<p><b>An update never needs a reboot — and it proves it took.</b> Before any file is replaced, the installer finds the bridge that is running — by its pid file, by the process holding her port, and by every bridge process working out of the install folder — and stops it <i>provably</i>: it waits for the process to be gone and the port to be free, escalates to a forced end if a polite signal is not enough, and if the port still cannot be cleared it aborts before touching a single file, naming the process that holds the port and the exact command that ends it (<span class='mono'>kill -9 &lt;pid&gt;</span>). Nothing is left nameless: a stuck process is a named process, with its remedy printed. If a bridge was running when the install began, the installer also relaunches her at the end and checks that the bridge answering reports the new build before it says <span class='mono'>Done.</span> — an update is not finished when the files land, but when the new build is the thing answering. The apply helper runs the same escalation before its own relaunch, and a dashboard left open across the update notices the bridge is newer than the page and offers a reload in the status column, so the new UI is one click away rather than one reboot. (An installer run that should leave her stopped can say so: <span class='mono'>--no-relaunch</span> — and it prints plainly that Apex is not running.)</p>" +
  "<p><b>Two channels: Stable and Edge.</b> The Updater follows one release channel, set by the <b>Release channel</b> card directly under the Current card in the Updater tab — the panel reads Current, then Release channel, then the patch notes, then the Check for updates / Download &amp; install buttons. <b>Stable</b> follows full releases only — cut from the project's main branch and tagged with the bare version number, like <b>v0.1.7</b>. <b>Edge</b> follows the newest release including pre-releases — test builds cut from the project's separate dev branch and tagged with an <span class='mono'>e</span>, like <b>v0.1.8e</b>. Edge is how you get a build before it is finished: newer, less settled. <b>Until you choose a channel yourself, the default follows the build you are running</b>: an Edge build defaults to the Edge channel, a full release to Stable — a preview is never silently stranded on a channel that does not carry it. A channel you pick in the panel is yours and it stays. And if you are running an Edge build and set the channel to Stable yourself, the Updater says so plainly, in a notice in the panel: Edge builds and refreshes of your preview are published only on Edge, so none will be offered while the channel stands. An Edge build of a version is the preview of that version, never an upgrade over its full release — when the full release of the same numbers lands, graduating to it is the update. Switching back to Stable never downgrades you; it simply waits for the next full release to pass what you run. The <b>About this release</b> dropdown in the Updater tab carries the release page of the version you are running — its name, tag, whether it is a full release or an Edge pre-release, its published date, the link to GitHub, and its full patch notes — fetched with the same check, straight from the release page.</p>" +
  "<p><b>Checks also happen in the background.</b> While the bridge runs, it re-checks your channel on its own every three hours, the first run shortly after startup, and the dashboard folds the verdict in quietly: an available update shows as <b>Update available</b> without you pressing anything, and a stale one clears. It never interrupts an install already under way. <b>Edge publications of one version are numbered</b> — Build 1, Build 2, and so on: the Updater's Current line shows the build you are running, and when the same Edge version is republished with fixes, the offer is a <b>refresh</b> of the preview you already run (Build 2 replaces Build 1), labeled with its build — not a new version. When the offered update is an Edge pre-release, an amber notice in the Updater says so plainly: preview contents can change between builds, your Soul, Memory, chips and packs carry over as always, and the next full release supersedes the preview automatically; graduating to that full release gets a green notice instead, because updating leaves the preview line.</p>" +
  "<p><b>Turn Sudo Mode on before you update.</b> The installer's system step needs root, and without it an update can stall or finish without its system packages — field-proven the hard way. So when an update is available and Sudo Mode is off, the Updater shows a red warning directly above the <b>Download &amp; install</b> button: <i>Enable Sudo Mode before updating.</i> Turn Sudo Mode on (in the status column) and the warning is replaced by a quiet green <i>Sudo Mode is on — ready to update.</i> It is a warning, not a lock — the button still works, because you may know your machine needs nothing new.</p>" +
  "<p><b>Once you opt in, the install chain moves.</b> The moment you press Download &amp; install, the step chain — downloading, verifying, staging, installing, DHD prerequisites, restarting — appears below the buttons, as the panel's tail, with a slim progress bar across the top of its card: the bar for the glance, the chain for the receipts. And if the Installing step runs long, the chain says why, in place: <i>Taking a while? This step is likely rebuilding the UI — that can take a few minutes on modest hardware. It's working, not stuck. We're working on making it faster.</i> No countdown and no ETA — on a modest machine an honest note beats a lying estimate.</p>" +
  "<p><b>One file says which version she is.</b> At the top of the install folder sits a plain text file named <span class='mono'>VERSION</span>. It holds the version number, and — only on an Edge build — the status word after it: <span class='mono'>0.2.0</span> is the current full release's own line; an Edge build reads like <span class='mono'>0.3.0 edge</span>. (The status word can be <span class='mono'>edge</span>, <span class='mono'>beta</span>, or <span class='mono'>stable</span>; no word means a full release. Builds from before the channels era wrote <span class='mono'>alpha</span> where Edge stands now — she reads those as Edge too, so nothing old is stranded.) It is the one source of truth: the bridge reads it and reports the number and the status, the update checker reads the same file to compare against the newest release on your channel, and the dashboard shows it at the bottom of the side menu (under a divider, centered) in GitHub tag form, as <b>v + number + a one-letter status tag</b> — Edge wears <span class='mono'>e</span>, beta wears <span class='mono'>b</span>, a full release wears none. So an Edge build reads <b>v0.3.0e</b> and this full release reads <b>v0.2.0</b>. With the side menu collapsed to its icon rail, the footer keeps the same v-form, centered, in the same status color. The number-and-tag is colored by status in every theme: red for Edge, yellow for beta, blue for a full release (in Nordic dark, Aurora red #BF616A, Aurora yellow #EBCB8B, and Frost blue #88C0D0). The status column's update entry carries no number and no label: it is a single line directly under Navi's status, bracketed by separators, reading a green <b>Up to date</b> once a check has run clean — and flipping to a yellow <b>Update available</b>, which opens the Updater, when a check finds a newer one. Nothing sits beneath the line. Installing an update replaces the VERSION file along with the rest of the app; no other place stores the version.</p></div>" },
  { id: "troubleshooting", title: "Troubleshooting", related: ["installing", "getting-started"], html:
    "<div class='doc-sec'><h4>Troubleshooting</h4><ul>" +
  "<li><b>Open Apex on your phone.</b> Put the phone on the <b>same network</b> as the machine Apex runs on. On the machine, open the <b>DHD</b> page and turn on <b>Apex | Node</b> [local], under the DHD options — that is the whole setup: Apex restarts her own bridge, and the row confirms <b>On your network</b> when it is live; there is no separate restart step. On the phone, open the address that same row shows (it looks like <span class='mono'>http://192.168.1.5:8675</span> — the row's button copies it) and type the <b>access code</b> from the row; the phone's browser is remembered after that. If the phone cannot reach her: check it is not on mobile data, check the row's live state line reads On your network, and check the code is the one the row shows now — regenerating the code shuts out every phone let in before. If the row names a failed restart step instead, press <b>Restart bridge</b> there, or quit Apex fully and relaunch. On the phone, the dashboard wears its small-screen layout: a slim bar stays pinned at the top — the hamburger button hugs its left edge with Apex | powered by Navi right beside it, the hamburger opening the menu (which slides in from the left), and the dock button hugging the right edge, opening the whole status column (which slides in from the right — the Chat window's own status toggle, at its top right, drives the same dock, one state behind two doors) — each a full-width overlay that closes itself when you pick where you're going. This machine itself never needs the code.</li>" +
  "<li><b>The dashboard shows some other page.</b> Something else holds port " + BRIDGE_PORT + " — the launcher names it by its page title; find it with <span class='mono'>ss -ltnp | grep " + BRIDGE_PORT + "</span>.</li>" +
  "<li><b>“BRIDGE OUT OF DATE”.</b> An older bridge process is still answering after a reinstall. The installer and the Updater now stop the old bridge and prove it stopped before new files land, so this should not survive an update; if you meet it after starting a bridge by hand, quit the app fully and relaunch — and if a process ever holds her port and will not stop, the failure that reports it names the process and the exact <span class='mono'>kill -9</span> command that ends it.</li>" +
  "<li><b>Chat 400s.</b> The bridge uses the model id the server itself reports (<span class='mono'>navi</span>); on failure the server's own error text is shown — check <span class='mono'>navi.llamafile.log</span>.</li>" +
  "<li><b>She doesn't answer.</b> Every chat request is bounded — at most 1,024 tokens of generation — and the bridge waits up to ten minutes for a CPU-bound reply to finish. If the wait runs out, the message in the thread says the model was still generating; that never means the server died. An empty reply from the model is reported the same way, in the thread, rather than swallowed as silence.</li>" +
  "<li><b>The llamafile won't start.</b> Press Connect — in app-launch mode the bridge starts the server itself and reports the specific failure.</li>" +
  "<li><b>Her voice is not installed.</b> Her voice is an extra the installer offers — it is not part of the main app. Say yes when the installer asks, or follow the step Settings → Voice names. Until then she still works; she just answers in writing instead of out loud.</li>" +
  "<li><b>A chip will not run.</b> Open it in Chip Studio and read the red squiggles first — a chip needs a <span class='mono'>command</span>, a <span class='mono'>use</span> line, and a <span class='mono'>run()</span> before Navi can run it, and the bar under the editor names what is missing. A chip that only declares an output is not broken: it shows that output instead of running code.</li>" +
  "<li><b>The catalog finds nothing.</b> The catalog is a live search of GitHub. With no network it finds nothing, and when GitHub itself is busy the search says so instead of pretending the shelf is empty — wait a minute and press Search again.</li>" +
  "<li><b>She answers without using my pack.</b> Only packs switched <b>on</b> feed her answers. Check the switch on the Packs tab. When a pack does help, the small label under her message names the pack and the document — no label means no pack was used for that answer.</li>" +
  "<li><b>The call will not start.</b> A call needs her voice installed and switched on, and her model connected. Settings → Voice names the missing piece — fix that one thing and try again.</li>" +
  "<li><b>Dark Liquid Glass looks solid, not frosted.</b> That is the design working as intended, not a bug: where your system cannot do backdrop blur, the theme falls back to the same colors, solid and fully readable. If scrolling ever feels stuttery on weak built-in graphics while the glass theme is on, pick any other theme in Settings → Appearance — the flat themes cost the graphics chip almost nothing.</li>" +
  "<li><b>Known issue: she cannot hear me in the desktop app.</b> Field report from the operator's Framework 13 (Pop!_OS), 2026-11-07, and noted in Settings → Voice at his request: in the standalone window, the microphone does not reach Navi — push-to-talk and calls get no working mic — even with the OS permission allowed. Her shell is built to grant the dashboard's mic requests; the field evidence — the window's own mic reading said granted, with no audio arriving — put the break below permission, in WebKit's media-capture settings, and the fix for that layer ships in <b>v0.4.1e</b>, compile-verified. It stays a known issue here until the operator's field bench confirms she hears him in the window. What works today: <b>typing</b>. The call panel takes typed lines and she answers out loud; Chat takes typed messages the whole time. If your mic works in the app, you are ahead of the field report — Settings → Voice shows what this window's own mic requests did this session, which helps narrow it down. Since v0.3.0e — Build 6, the window also <b>reports that reading to the bridge on every push-to-talk attempt</b> — it lands in the bridge's log (start.log) with the attempt, so the evidence accumulates in the open; it is the record the v0.4.1e fix was aimed from.</li></ul></div>" },
  { id: "glossary", title: "Glossary — the words we use", related: ["getting-started", "chips-guide"], html:
    "<div class='doc-sec'><h4>Glossary — the words we use, in plain words</h4>" +
  "<p>Every special word in this wiki, explained once, in the order you are most likely to meet it. If a page ever uses a word that is not clear, it should be in this list.</p>" +
  "<p><b>Dashboard</b> — the page you are looking at. The buttons, the chat, the lists: all of it is the dashboard.</p>" +
  "<p><b>Bridge</b> — a small helper program that lives in the install folder. The dashboard asks it to do things — run a command, start the model, save a setting — and it does them.</p>" +
  "<p><b>Model</b> — the part that thinks and writes her answers. Hers lives inside one big file.</p>" +
  "<p><b>navi.llamafile / llamafile</b> — that big file. It holds the model and the program that runs it, all in one file, so there is nothing else to install.</p>" +
  "<p><b>Port</b> — a numbered door on your own computer that programs talk through. The bridge listens at door " + BRIDGE_PORT + "; the model at door 8080. Nothing from the internet comes through them.</p>" +
  "<p><b>Chip</b> — a small program that gives Navi a new skill. You write it, or install one somebody else wrote.</p>" +
  "<p><b>Chip</b> — a chip packed into one picture file. The code hides inside the image. Share the picture and you have shared the chip.</p>" +
  "<p><b>root.py</b> — a chip's main file, the one Navi runs. Every chip has exactly one. Other files are helpers it can bring in with <span class='mono'>import</span>.</p>" +
  "<p><b>Workflow</b> — chips lined up in a row, where each one's answer is handed to the next one, like a bucket brigade.</p>" +
  "<p><b>Pack</b> — a bundle of documents Navi can look things up in before she answers. One pack is one topic.</p>" +
  "<p><b>Index</b> — the map a pack builds from its documents, so Navi can find the right page fast instead of reading everything every time.</p>" +
  "<p><b>Active</b> — switched on. Only active packs feed her answers.</p>" +
  "<p><b>Repository (repo)</b> — one project's folder on GitHub, the website that holds code.</p>" +
  "<p><b>Topic</b> — a label on a repository. The label <span class='mono'>navi-chips</span> is how Navi finds chips; <span class='mono'>navi-rag</span> is how she finds packs.</p>" +
  "<p><b>Release / tag</b> — a release is GitHub's way of saying “this exact version, ready to download”. Its tag is the version's name label, like <span class='mono'>0.1.0</span>. A bigger tag than the one you have is what offers an update.</p>" +
  "<p><b>Manifest</b> — a small card inside an export (like <span class='mono'>navi-chip.json</span>) that says what the package is: its name, its version, and which file starts it.</p>" +
  "<p><b>Soul / Memory</b> — two plain documents in the config folder: who Navi is, and what she carries with her. Together they are her <b>persona</b> — the way she acts.</p>" +
  "<p><b>Sudo / root</b> — <b>root</b> is the master account that can do anything on the machine. <b>Sudo</b> is asking to borrow it for one job.</p>" +
  "<p><b>Backend</b> — the engine the dashboard talks to. In the shipped app it is always <b>Live</b>: the real bridge and model. (The Apex test bench in the space keeps its own labeled Mock/Live switch for testing; that switch is not part of this app.)</p>" +
  "<p><b>Catalog</b> — the list the Chips and RAG tabs build by searching GitHub. It is a search result, not a store — the chips and packs themselves live in their repositories.</p></div>" },
];

/* ------------------------------ live calls ------------------------------ */
function stripCite(text) {
  // Citations are presentation, not prose: sources ride the chat
  // payload and render as pills under the message (srcPills). A
  // footer in the old shape ("— from …") is stripped wherever one
  // survives — a thread stored by an older build — so it never
  // shows as part of her words, never gets spoken, and never
  // rides back to the model as history.
  var lines = String(text || "").replace(/\s+$/, "").split("\n");
  while (lines.length && lines[lines.length - 1].trim().indexOf("— from ") === 0) {
    lines.pop();
    while (lines.length && !lines[lines.length - 1].trim()) lines.pop();
  }
  return lines.join("\n");
}
/* --------------------- memory from chat (build .81, #72) --------------------- */
// The dashboard half of memory-from-chat: the bridge decides
// (trigger extraction, and the model's [[remember:]] markers
// stripped into payload data); the filing happens here, against
// the local store, with the correction rule — a save about the
// same fact as an existing entry replaces it (matched by fact
// key or heavy word overlap), an exact repeat is "already", and
// every outcome becomes a receipt on the message. In mock mode
// the same analysis runs in-page (jsMemoryTrigger /
// jsMemoryAutonomous mirror the bridge's rules).
var MEMORY_SECRET_RE = /\b(password|passcode|passwd|api[\s_-]?key|secret|token|credential|ssn|social security|credit card|card number|cvv)\b/i;
var MEMORY_LONG_DIGITS_RE = /\b(?:\d[ -]?){13,19}\b/;
function jsMemorySecret(text) { return MEMORY_SECRET_RE.test(text || "") || MEMORY_LONG_DIGITS_RE.test(text || ""); }
function jsNormalizeFact(text) {
  var t = String(text || "").replace(/\s+/g, " ").trim().replace(/^["']|["']$/g, "");
  if (!t) return "";
  t = t.charAt(0).toUpperCase() + t.slice(1);
  return /[.!?]$/.test(t) ? t : t + ".";
}
function jsFactKey(text) {
  var t = String(text || "").toLowerCase();
  if (/(?:operator'?s|my) name is\b/.test(t) || /^name\s*:/.test(t)) return "name";
  if (/(?:operator lives|i live|i'?m based|operator'?s (?:home|hometown) is|my (?:home|hometown) is)\b/.test(t)) return "home";
  if (/(?:operator works|i work|my job is|operator'?s job is)\b/.test(t)) return "work";
  var fav = t.match(/favou?rite\s+([a-z][a-z -]*?)\s+is\b/);
  if (fav && fav[1]) return "favorite:" + fav[1].trim();
  return null;
}
var MEMORY_STOPWORDS = { a: 1, an: 1, the: 1, is: 1, are: 1, was: 1, were: 1, my: 1, i: 1, to: 1, in: 1, at: 1, for: 1, of: 1, on: 1, and: 1, or: 1, that: 1, this: 1, it: 1, its: 1, operator: 1, operators: 1, s: 1, me: 1, we: 1, you: 1 };
function jsWordSet(text) {
  var out = {};
  String(text || "").toLowerCase().replace(/[^a-z0-9 ]+/g, " ").split(/\s+/).forEach(function (w) { if (w.length >= 3 && !MEMORY_STOPWORDS[w]) out[w] = 1; });
  return out;
}
function jsSameFact(a, b) {
  var ka = jsFactKey(a), kb = jsFactKey(b);
  if (ka && kb) return ka === kb;
  var sa = jsWordSet(a), sb = jsWordSet(b), ak = Object.keys(sa), bk = Object.keys(sb);
  if (!ak.length || !bk.length) return false;
  var shared = 0;
  ak.forEach(function (w) { if (sb[w]) shared++; });
  return shared / Math.max(ak.length, bk.length) >= 0.55;
}
function jsMemoryTrigger(raw) {
  var line = String(raw || "").trim().replace(/^>\s*/, "");
  if (/\?/.test(line) && /^(?:do|did|can|could|what|why|how|remember when)\b/i.test(line)) return null;
  var saveTo = line.match(/^(?:please\s+)?(?:navi[,\s]+)?(?:save|write|put)\s+(?:this|that)\s+(?:to|in|into)\s+(?:your\s+)?memory\s*[:-]?\s*(.*)$/i);
  if (saveTo) { var r0 = (saveTo[1] || "").trim(); return r0 ? { kind: "fact", fact: r0 } : { kind: "prior" }; }
  var m = line.match(/^(?:please\s+)?(?:navi[,\s]+)?remember[\s:]+(.*)$/i);
  if (!m) return null;
  var rest = (m[1] || "").trim();
  if (!rest) return null;
  if (/^(?:when|me|us|the time)\b/i.test(rest)) return null;
  var pointed = rest.match(/^(?:this|that)\b\s*[:-—,]?\s*(.*)$/i);
  if (pointed) { rest = (pointed[1] || "").trim(); return rest ? { kind: "fact", fact: rest } : { kind: "prior" }; }
  if (rest.charAt(0) === ":" || rest.charAt(0) === "-") { rest = rest.slice(1).trim(); return rest ? { kind: "fact", fact: rest } : { kind: "prior" }; }
  return { kind: "fact", fact: rest };
}
function jsMemoryAutonomous(raw) {
  var line = String(raw || "").trim().replace(/^>\s*/, "");
  if (!line || line.length < 10) return null;
  if (line.charAt(line.length - 1) === "?") return null;
  if (/\bremember\b/i.test(line)) return null;
  function cap(s) { return s.trim().replace(/\s+/g, " "); }
  var m = line.match(/\bmy name is\s+([^.,;!?]+)/i);
  if (m && m[1]) return jsNormalizeFact("Operator's name is " + cap(m[1]));
  m = line.match(/\bi (?:live|am living) in\s+([^.,;!?]+)/i) || line.match(/\bi'?m based in\s+([^.,;!?]+)/i);
  if (m && m[1]) return jsNormalizeFact("Operator lives in " + cap(m[1]));
  m = line.match(/\bi work\s+(at|for|as)\s+([^.,;!?]+)/i);
  if (m && m[1] && m[2]) return jsNormalizeFact("Operator works " + m[1].toLowerCase() + " " + cap(m[2]));
  m = line.match(/\bmy job is\s+([^.,;!?]+)/i);
  if (m && m[1]) return jsNormalizeFact("Operator's job is " + cap(m[1]));
  m = line.match(/\bmy favou?rite\s+([a-z][a-z -]*?)\s+is\s+([^.,;!?]+)/i);
  if (m && m[1] && m[2]) return jsNormalizeFact("Operator's favorite " + cap(m[1]).toLowerCase() + " is " + cap(m[2]));
  if (/\bi (?:prefer|like|love|hate|dislike)\b/i.test(line) || /\bi don'?t (?:like|want)\b/i.test(line) || /\bwe (?:decided|agreed)\b/i.test(line)) return jsNormalizeFact(line.replace(/[.!?]+$/, ""));
  return null;
}
function jsMemoryCapture(text) {
  // Mock-mode autonomous capture (the live path's judgment is
  // the model's, via the bridge). Returns the payload shape
  // sendChat's done() applies, or null.
  if ((CONFIG.memoryMode || "autonomous") === "trigger") return null;
  var fact = jsMemoryAutonomous(text);
  if (!fact) return null;
  if (jsMemorySecret(fact) || jsMemorySecret(text)) return { saves: null, refused: true };
  return { saves: [{ text: fact, origin: "autonomous", proposed: CONFIG.memoryMode === "ask" }], refused: false };
}
function applyMemorySave(save) {
  var text = save.text;
  var norm = text.toLowerCase().replace(/\s+/g, " ").trim();
  for (var i = 0; i < memory.length; i++) {
    if (memory[i].t.toLowerCase().replace(/\s+/g, " ").trim() === norm) return { kind: "already", text: memory[i].t, replacedText: null, origin: save.origin, undone: false, proposalId: null };
  }
  for (var j = 0; j < memory.length; j++) {
    if (jsSameFact(text, memory[j].t)) {
      var replaced = memory[j].t;
      memory[j] = { t: text, o: "correction" };
      persist();
      return { kind: "corrected", text: text, replacedText: replaced, origin: save.origin, undone: false, proposalId: null };
    }
  }
  memory.push({ t: text, o: save.origin === "trigger" ? "trigger" : "autonomous" });
  persist();
  return { kind: "saved", text: text, replacedText: null, origin: save.origin, undone: false, proposalId: null };
}
function applyMemorySaves(saves, refused) {
  var receipts = [];
  (saves || []).forEach(function (s) {
    if (s.proposed) {
      var p = { id: "memprop-" + Date.now() + "-" + Math.floor(Math.random() * 1e6), text: s.text, origin: s.origin || "autonomous" };
      memoryProposals.push(p);
      receipts.push({ kind: "proposed", text: s.text, replacedText: null, origin: p.origin, undone: false, proposalId: p.id });
    } else {
      receipts.push(applyMemorySave(s));
    }
  });
  persistMemoryProposals();
  if (refused) receipts.push({ kind: "refused", text: "", replacedText: null, origin: "autonomous", undone: false, proposalId: null });
  return receipts;
}
function undoMemoryReceipt(rc) {
  if (rc.kind === "saved") {
    var i = memory.findIndex(function (mm) { return mm.t === rc.text; });
    if (i >= 0) { seatForgotten.push(memory[i].t); memory.splice(i, 1); }
  } else if (rc.kind === "corrected") {
    var j = memory.findIndex(function (mm) { return mm.t === rc.text; });
    if (j >= 0 && rc.replacedText) memory[j] = { t: rc.replacedText, o: "manual" };
  } else if (rc.kind === "proposed" && rc.proposalId) {
    memoryProposals = memoryProposals.filter(function (p) { return p.id !== rc.proposalId; });
    persistMemoryProposals();
  } else return false;
  rc.undone = true;
  persist();
  return true;
}
function resolveMemoryProposal(rc, accept) {
  if (!rc.proposalId) return;
  var p = memoryProposals.filter(function (x) { return x.id === rc.proposalId; })[0];
  memoryProposals = memoryProposals.filter(function (x) { return x.id !== rc.proposalId; });
  persistMemoryProposals();
  if (accept && p) {
    var r = applyMemorySave({ text: p.text, origin: p.origin });
    rc.kind = r.kind; rc.text = r.text; rc.replacedText = r.replacedText; rc.proposalId = null;
  } else if (!accept) { rc.kind = "dismissed"; rc.proposalId = null; }
  persist();
}
function resolveProposalEverywhere(pid, accept) {
  var found = null;
  chats.forEach(function (c) { c.messages.forEach(function (m) { (m.memoryReceipts || []).forEach(function (rc) { if (rc.proposalId === pid) found = rc; }); }); });
  if (found) resolveMemoryProposal(found, accept);
  else {
    var p = memoryProposals.filter(function (x) { return x.id === pid; })[0];
    memoryProposals = memoryProposals.filter(function (x) { return x.id !== pid; });
    persistMemoryProposals();
    if (accept && p) applyMemorySave({ text: p.text, origin: p.origin });
  }
  persist(); render();
}
function memoryReceiptHtml(m, idx) {
  if (!m.memoryReceipts || !m.memoryReceipts.length) return "";
  return m.memoryReceipts.map(function (rc, ri) {
    var head = rc.kind === "corrected" ? "Memory updated" : rc.kind === "already" ? "Already remembered" : rc.kind === "proposed" ? "Navi wants to remember this" : rc.kind === "refused" ? "Not saved" : rc.kind === "dismissed" ? "Memory proposal dismissed" : "Saved to memory";
    var originLine = rc.origin === "trigger" ? "From chat — you asked" : "From chat — her call";
    var h = "<div class='card' style='margin-top:8px;padding:8px 10px" + (rc.kind === "refused" ? ";border-color:#f87171" : "") + "'><div class='mono' style='font-size:12px'><b>" + head + "</b> <span style='color:var(--dim)'>" + (rc.kind === "refused" ? "secret refused" : originLine) + (rc.undone ? " · undone" : "") + "</span></div>";
    if (rc.kind === "refused") {
      h += "<div style='color:var(--dim);font-size:13px;margin-top:4px'>That looked like a secret — a password, a key, or a card number — and secrets never go into her memory, on any path.</div>";
    } else {
      if (rc.kind === "corrected" && rc.replacedText) h += "<div style='color:var(--dim);font-size:13px;margin-top:4px'><s>" + esc(rc.replacedText) + "</s></div>";
      h += "<div style='font-size:13px;margin-top:2px" + ((rc.kind === "dismissed" || rc.undone) ? ";opacity:.6" : "") + "'>“" + esc(rc.text) + "”</div>";
    }
    if (!rc.undone && (rc.kind === "saved" || rc.kind === "corrected")) {
      h += "<div style='margin-top:6px'><button class='ghost' data-memundo='" + idx + ":" + ri + "' aria-label='Undo this memory save' style='padding:1px 8px'>" + (rc.kind === "corrected" ? "Undo — restore the old line" : "Undo") + "</button></div>";
    }
    if (!rc.undone && rc.kind === "proposed") {
      h += "<div style='margin-top:6px'><button class='act' data-memsave='" + idx + ":" + ri + "' aria-label='Save this memory proposal' style='padding:2px 10px'>Save it</button> <button class='ghost' data-memdismiss='" + idx + ":" + ri + "' aria-label='Dismiss this memory proposal' style='padding:2px 10px'>Dismiss</button></div>";
    }
    return h + "</div>";
  }).join("");
}
function caseLogHtml(m) {
  // Build 2 (#199): the receipt for Navi's case logging — what
  // she wrote into the case's registers from this reply, every
  // entry badged UNVERIFIED (it lands as a lead until the
  // operator reviews it in Case Files), plus the cross-case
  // flags: a detail that matches another case is NAMED here
  // and never written across.
  if (!m.caseLog) return "";
  var h = "";
  var entries = m.caseLog.entries || [];
  if (entries.length) {
    h += "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Logged to the case</b> <span style='color:var(--am)'>unverified — review in Case Files</span></div>"
      + entries.map(function (e) {
        return "<div style='font-size:13px;margin-top:2px'><span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(e.kind.toUpperCase()) + "</span> “" + esc(e.label) + "” <span class='mono' style='font-size:11px;color:var(--am)'>LEAD</span></div>";
      }).join("") + "</div>";
  }
  (m.caseLog.flags || []).forEach(function (f) {
    h += "<div class='card' style='margin-top:8px;padding:8px 10px;border-color:var(--am)'><div style='font-size:13px'><b>Cross-case detail:</b> “" + esc(f.value) + "” also appears in <b>" + esc(f.caseName || f.case) + "</b> — flagged here, not written across. Mention it to Navi if the cases connect.</div></div>";
  });
  // v0.5.2e (#261): the grounding flags and the named refusals
  // ride the same receipt — a filing that contradicted its
  // Wire record is flagged on its face, and a malformed filing
  // attempt is refused by name, never silently dropped.
  (m.caseLog.grounding || []).forEach(function (g) {
    h += "<div class='card' style='margin-top:8px;padding:8px 10px;border-color:var(--am)'><div style='font-size:13px'><b>Grounding flag:</b> this filing states " + esc(g.field) + " " + esc(g.claimed) + " for " + esc(g.cve) + " — the Wire record says " + esc(g.field) + " " + esc(g.record) + ". Filed as written; the record's values govern.</div></div>";
  });
  (m.caseLog.refusals || []).forEach(function (r) {
    h += "<div class='card' style='margin-top:8px;padding:8px 10px;border-color:#f87171'><div class='mono' style='font-size:12px'><b>Filing did not land</b></div><div style='font-size:13px;margin-top:2px'>" + esc(r) + "</div></div>";
  });
  return h;
}
function caseFiledReplyHtml(m) {
  // v0.5.2e (#261, leg 1): the receipt for the deterministic
  // door — a reply the operator filed into its case with the
  // "File into this case" tap. The bridge appended this
  // message itself, from what it actually did; the card
  // carries the finding's path and any grounding flag.
  var f = m.caseFiledReply;
  if (!f) return "";
  var h = "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Filed to the case</b> <span style='color:var(--dim)'>by your tap · receipted by the bridge</span></div>"
    + "<div style='font-size:13px;margin-top:2px'>Into <b>" + esc(f.caseName || f.case) + "</b> " + (f.part === "task" ? "as tasks (" + (f.count || 1) + ")" : "as " + casePartArticle(f.part || "finding") + " " + esc(f.part || "finding")) + "</div>"
    + "<div class='mono' style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(f.path) + "</div>";
  (f.grounding || []).forEach(function (g) {
    h += "<div style='font-size:12px;color:var(--am);margin-top:2px'>Grounding flag: the reply states " + esc(g.field) + " " + esc(g.claimed) + " for " + esc(g.cve) + " — the Wire record says " + esc(g.record) + ". Filed as written; the record's values govern.</div>";
  });
  return h + "</div>";
}
function casePartArticle(part) { return (part === "observable" || part === "entity" || part === "event" || part === "evidence") ? "an" : "a"; }
// v0.6.6e (#327): the page-side mirror of the bridge's
// case_suggest_detail and its helpers — the same
// deterministic read, helper for helper, so the button's
// named destination and the bridge's landing can never
// disagree. She senses the shape: a chronological narrative
// is a timeline, a list of indicator data is observables, an
// imperative next-step list is tasks, a short typed subject
// is an entity, a reply opening on a date is an event, a
// reply carrying its sourcing is a lead — and anything else
// is the honest plain default, a finding. The operator's
// words naming a part govern over the sense on every door.
var CASE_DATE_SRC = "(?:\\d{4}-\\d{2}-\\d{2}\\b|(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]*\\.?\\s+\\d{1,2},?\\s+\\d{4}\\b|\\d{1,2}\\s+(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]*\\.?\\s+\\d{4}\\b)";
var CASE_DATE_RE = new RegExp("\\b" + CASE_DATE_SRC, "i");
var CASE_DATE_START_RE = new RegExp("^" + CASE_DATE_SRC, "i");
var CASE_MONTHS = { jan: 1, feb: 2, mar: 3, apr: 4, may: 5, jun: 6, jul: 7, aug: 8, sep: 9, oct: 10, nov: 11, dec: 12 };
var CASE_TASK_VERBS = {};
("pull check review patch update verify confirm contact interview search scan collect file send draft write run test audit trace map monitor follow ask get fetch dump extract examine inspect document schedule deploy fix build create add remove escalate notify preserve acquire hunt triage correlate enumerate identify isolate contain flag export import compare watch read open close rotate revoke reset enable disable block quarantine snapshot capture record log ping call book sweep parse enrich pivot brief stage").split(" ").forEach(function (v) { CASE_TASK_VERBS[v] = 1; });
var CASE_ENTITY_TYPES = "(?:person|organi[sz]ation|org|company|vendor|account|system|place|service|tool|actor|group|host|server|domain|wallet|exchange)";
function caseStripTok(w) { return String(w || "").replace(/^[`*,.:;()\[\]<>"'\u2014-]+/, "").replace(/[`*,.:;()\[\]<>"'\u2014-]+$/, ""); }
function caseNamedDate(text) {
  var m = new RegExp("\\b(\\d{4})-(\\d{2})-(\\d{2})\\b|\\b(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]*\\.?\\s+(\\d{1,2}),?\\s+(\\d{4})\\b|\\b(\\d{1,2})\\s+(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]*\\.?\\s+(\\d{4})\\b", "i").exec(String(text || ""));
  if (!m) return null;
  var y, mo, d;
  if (m[1]) { y = +m[1]; mo = +m[2]; d = +m[3]; }
  else if (m[4]) { mo = CASE_MONTHS[m[4].toLowerCase().slice(0, 3)]; d = +m[5]; y = +m[6]; }
  else { d = +m[7]; mo = CASE_MONTHS[m[8].toLowerCase().slice(0, 3)]; y = +m[9]; }
  var dt = new Date(Date.UTC(y, mo - 1, d));
  if (dt.getUTCFullYear() !== y || dt.getUTCMonth() !== mo - 1 || dt.getUTCDate() !== d) return null;
  function p2(n) { return (n < 10 ? "0" : "") + n; }
  return y + "-" + p2(mo) + "-" + p2(d);
}
function caseListItems(text) {
  var items = [], prose = [];
  String(text || "").split("\n").forEach(function (ln) {
    var s = ln.trim();
    if (!s) return;
    var m = /^(?:[-*\u2022]|\d{1,3}[.)])\s+(.*)$/.exec(s);
    if (m && m[1].trim()) items.push(m[1].trim()); else prose.push(s);
  });
  return { items: items, prose: prose };
}
function caseIndicatorToken(tok) {
  var w = caseStripTok(tok);
  return /^\d{1,3}(\.\d{1,3}){3}(:\d+)?$/.test(w)
    || /^[a-fA-F0-9]{32,64}$/.test(w)
    || /^CVE-\d{4}-\d{4,}$/i.test(w)
    || /^[\w.\-]+@[\w.\-]+\.\w+$/.test(w)
    || /^https?:\/\/\S+$/.test(w)
    || /^[\w\-]+(\.[\w\-]+){2,}(\/\S*)?$/.test(w);
}
function caseIndicatorLine(line) {
  var toks = String(line || "").split(/\s+/).filter(Boolean);
  if (!toks.length) return false;
  if (caseIndicatorToken(toks[0])) return true;
  var hits = 0;
  toks.forEach(function (tk) { if (caseIndicatorToken(tk)) hits++; });
  return hits * 5 >= toks.length * 3;
}
function caseImperativeItem(item) {
  var toks = String(item || "").trim().replace(/^[`*_]+/, "").replace(/[`*_]+$/, "").split(/\s+/).filter(Boolean);
  if (!toks.length) return false;
  var first = caseStripTok(toks[0]).toLowerCase();
  if (first === "to" && toks.length > 1) first = caseStripTok(toks[1]).toLowerCase();
  return !!CASE_TASK_VERBS[first];
}
function caseTaskSplit(text) {
  var sp = caseListItems(text);
  if (sp.items.length) return sp;
  var lines = String(text || "").split("\n").map(function (s) { return s.trim(); }).filter(Boolean);
  if (!lines.length) return { items: [], prose: [] };
  var head = lines[0].replace(/#/g, " ").split(/\s+/).filter(Boolean).join(" ").replace(/^[ .:—-]+/, "").replace(/[ .:—-]+$/, "");
  return { items: head ? [head] : [], prose: lines.slice(1) };
}
function caseSuggestDetail(text) {
  var t = String(text || "").trim();
  if (!t) return { part: "finding", sensed: false };
  var lines = t.split("\n").map(function (s) { return s.trim(); }).filter(Boolean);
  var dated = lines.filter(function (ln) { return CASE_DATE_RE.test(ln); });
  if (dated.length >= 3) return { part: "timeline", sensed: true };
  var low0 = t.toLowerCase();
  var sourced = /\baccording to\b/.test(low0)
    || /\b(reports?|advisory|bulletin|record|sources?|writeup|post)\s+(says?|said|reports?|shows?|states?|notes?|found)\b/.test(low0)
    || /\]\((https?:\/\/|www\.)/.test(t)
    || /\bis reported(ly)?\b/.test(low0)
    || /\bclaims?\s+that\b/.test(low0);
  var sp = caseListItems(t);
  if (sp.items.length >= 2 && !sourced) {
    var ind = 0, imp = 0;
    sp.items.forEach(function (it) { if (caseIndicatorLine(it)) ind++; if (caseImperativeItem(it)) imp++; });
    if (ind * 2 >= sp.items.length) return { part: "observable", sensed: true };
    var leadin = sp.prose.some(function (ln) { return /\bnext steps?\b|\btasks?\b|\bto[- ]do\b/i.test(ln); });
    if (leadin || imp * 2 >= sp.items.length) return { part: "task", sensed: true };
  }
  var words = t.split(/\s+/).filter(Boolean);
  if (words.length <= 6) {
    for (var i = 0; i < words.length; i++) if (caseIndicatorToken(words[i])) return { part: "observable", sensed: true };
  }
  if (words.length <= 30 && lines.length <= 2) {
    if (new RegExp("\\bis\\s+an?\\s+" + CASE_ENTITY_TYPES + "\\b", "i").test(t)
        || new RegExp("\\s[\u2014\u2013-]\\s+an?\\s+" + CASE_ENTITY_TYPES + "\\b", "i").test(t)
        || new RegExp("\\s[\u2014\u2013-]\\s+" + CASE_ENTITY_TYPES + "\\s*$", "i").test(t)) return { part: "entity", sensed: true };
  }
  if (lines.length && (CASE_DATE_START_RE.test(lines[0])
      || /^on\s+(the\s+)?\d/i.test(lines[0])
      || (caseNamedDate(lines[0]) && lines[0].toLowerCase().indexOf("on ") === 0))) return { part: "event", sensed: true };
  if (sourced) return { part: "lead", sensed: true };
  return { part: "finding", sensed: false };
}
function caseSuggestPart(text) { return caseSuggestDetail(text).part; }
function casePartDestLabel(part, text) {
  // The destination as the button names it (#326/#327): a
  // task filing names its count from the same split the
  // bridge lands; the other new parts name their section.
  if (part === "task") { var n = caseTaskSplit(text).items.length; return n > 1 ? "as tasks (" + n + ")" : "as a task"; }
  if (part === "timeline") return "as a timeline entry";
  if (part === "evidence") return "as evidence";
  if (part === "report") return "as report material";
  return "as " + casePartArticle(part) + " " + part;
}
function caseFileReplyBtnHtml(m, idx) {
  // The tap itself: on Navi's replies, only while the open
  // chat is bound to a case, and never on the bridge's own
  // receipt messages (filing a receipt would file nothing new).
  if (!m.text || m.caseFiledReply) return "";
  var ac = (typeof activeChat === "function") ? activeChat() : null;
  if (!ac || !ac.case) return "";
  var part = caseSuggestPart(m.text);
  var dest = casePartDestLabel(part, m.text);
  return "<div style='margin-top:6px'><button class='ghost' data-file-reply='" + idx + "' data-file-part='" + part + "' aria-label='File this reply into the bound case " + dest + "' style='min-height:36px;padding:4px 12px;font-size:12px'>File into this case — " + dest + "</button></div>";
}
function caseStartedHtml(m) {
  // v0.5.14e (#289): the receipt for the "Start a case from
  // this" door — a case the operator started from one of her
  // replies by pressing its button. The bridge appended this
  // message itself, from what it actually did; the card
  // carries the case's name and the finding's path, any
  // grounding flag, and the door into the case's own chat —
  // the standing intake-receipt button wiring
  // (data-caseintake-open) walks it, unchanged.
  var s = m.caseStarted;
  if (!s) return "";
  var h = "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Case started</b> <span style='color:var(--dim)'>by your tap \u00b7 receipted by the bridge</span></div>"
    + "<div style='font-size:13px;margin-top:2px'><b>" + esc(s.name || s.slug) + "</b> (" + esc(s.type || "investigation") + ") \u2014 " + (!s.part || s.part === "finding" ? "the reply stands in its findings, in her name" : (s.part === "event" || s.part === "timeline" ? "the reply stands on its timeline, in her name" : (s.part === "task" ? "the reply stands in its tasks, in her name" : (s.part === "evidence" ? "the reply stands in its evidence, in her name" : (s.part === "report" ? "the reply stands in its report material, in her name" : "the reply stands in the case as " + casePartArticle(s.part) + " " + esc(s.part) + ", in her name"))))) + "</div>"
    + "<div class='mono' style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(s.path || "") + "</div>";
  (s.grounding || []).forEach(function (g) {
    h += "<div style='font-size:12px;color:var(--am);margin-top:2px'>Grounding flag: the reply states " + esc(g.field) + " " + esc(g.claimed) + " for " + esc(g.cve) + " \u2014 the Wire record says " + esc(g.record) + ". Filed as written; the record's values govern.</div>";
  });
  if (s.chatId) h += "<div style='margin-top:8px'><button class='act' data-caseintake-open='" + esc(String(s.chatId)) + "' data-caseintake-slug='" + esc(s.slug || "") + "' aria-label='Open the case chat for " + esc(s.name || "the new case") + "' style='min-height:44px;padding:6px 14px'>Open the case chat \u2014 " + esc(s.name || "the case") + "</button></div>";
  return h + "</div>";
}
function caseStartBtnHtml(m, idx) {
  // v0.5.14e (#289): the standing offer — on Navi's
  // replies in a chat that is NOT a case room (the mirror of
  // the filing tap, which stands in case rooms). Never on the
  // bridge's own receipt messages, and never on her tray
  // announcements: a notification is not case material.
  if (!m.text || m.caseFiledReply || m.caseStarted || m.notice) return "";
  var ac = (typeof activeChat === "function") ? activeChat() : null;
  if (!ac || ac.case) return "";
  var spart = caseSuggestPart(m.text);
  var sdest = casePartDestLabel(spart, m.text);
  return "<div style='margin-top:6px'><button class='ghost' data-start-case='" + idx + "' data-start-part='" + spart + "' aria-label='Start a case from this reply, filed " + sdest + "' style='min-height:36px;padding:4px 12px;font-size:12px'>Start a case from this — " + sdest + "</button></div>";
}
function caseFilingsHtml(m) {
  // Build 15 (#161): the receipt for Navi's own case filings —
  // what she filed, into which case, at which path. Filed in
  // her name; the finding itself lives in the case's findings/.
  if (!m.caseFilings || !m.caseFilings.length) return "";
  return m.caseFilings.map(function (f) {
    return "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Filed to the case</b> <span style='color:var(--dim)'>her finding, in her name</span></div>"
      + "<div style='font-size:13px;margin-top:2px'>“" + esc(f.title) + "” — into <b>" + esc(f.caseName || f.case) + "</b></div>"
      + "<div class='mono' style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(f.path) + "</div>"
      + (f.grounding && f.grounding.length ? f.grounding.map(function (g) { return "<div style='font-size:12px;color:var(--am);margin-top:2px'>Grounding flag: states " + esc(g.field) + " " + esc(g.claimed) + " for " + esc(g.cve) + " — the Wire record says " + esc(g.record) + ". The record's values govern.</div>"; }).join("") : "")
      + "</div>";
  }).join("");
}
function caseIntakeHtml(m) {
  // v0.4.5e (#241): the receipt for a case started from chat —
  // built from the BRIDGE's record of what it did, never from
  // her prose. A born case gets its counts and the door into
  // its own chat: the button rides the receipt's chatId and
  // walks the same /api/cases/chat jump the case page uses.
  // No chatId, no button — a button that does nothing is
  // never rendered. A refusal renders as a named card with no
  // button at all.
  var it = m.caseIntake;
  if (!it) return "";
  if (it.ok === false) {
    return "<div class='card' style='margin-top:8px;padding:8px 10px;border-color:var(--am)'><div class='mono' style='font-size:12px'><b>Case not started</b></div>"
      + "<div style='font-size:13px;margin-top:2px'>" + esc(it.error || "The intake could not be read.") + " Nothing was created — the <b>New case</b> button beside the composer will pre-fill a form from this conversation instead.</div></div>";
  }
  var bits = [];
  if (it.findings) bits.push(it.findings + (it.findings === 1 ? " finding" : " findings") + " filed");
  if (it.entities) bits.push(it.entities + (it.entities === 1 ? " entity" : " entities"));
  if (it.observables) bits.push(it.observables + (it.observables === 1 ? " observable" : " observables"));
  if (it.events) bits.push(it.events + (it.events === 1 ? " timeline beat" : " timeline beats"));
  if (it.tasks) bits.push(it.tasks + (it.tasks === 1 ? " task" : " tasks") + " proposed");
  var h = "<div class='card' style='margin-top:8px;padding:8px 10px'><div class='mono' style='font-size:12px'><b>Case opened</b> <span style='color:var(--dim)'>from this chat · set as the open case</span></div>"
    + "<div style='font-size:13px;margin-top:2px'><b>" + esc(it.name || it.slug || "Case") + "</b>" + (it.type ? " <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(it.type.toUpperCase()) + "</span>" : "") + "</div>"
    + (bits.length ? "<div style='font-size:13px;margin-top:2px'>" + esc(bits.join(" · ")) + ((it.entities || it.observables) ? " <span style='color:var(--am)'>— logged unverified, review in Case Files</span>" : "") + "</div>" : "")
    + (it.dropped ? "<div style='font-size:12px;color:var(--am);margin-top:2px'>" + it.dropped + (it.dropped === 1 ? " further item" : " further items") + " did not fit the intake caps and were dropped — named, not silent.</div>" : "");
  (it.grounding || []).forEach(function (g) {
    h += "<div style='font-size:12px;color:var(--am);margin-top:2px'>Grounding flag: a filed finding states " + esc(g.field) + " " + esc(g.claimed) + " for " + esc(g.cve) + " — the Wire record says " + esc(g.record) + ". The flag rides the finding; the record's values govern.</div>";
  });
  if (it.chatId) {
    h += "<div style='margin-top:8px'><button class='act' data-caseintake-open='" + esc(String(it.chatId)) + "' data-caseintake-slug='" + esc(it.slug || "") + "' aria-label='Open the case chat for " + esc(it.name || "the new case") + "' style='min-height:44px;padding:6px 14px'>Open the case chat — " + esc(it.name || "the case") + "</button></div>";
  }
  return h + "</div>";
}
var MEMORY_ORIGIN_LABELS = { manual: "Added by hand", trigger: "From chat — you asked", autonomous: "From chat — her call", correction: "From chat — correction" };
function srcPills(sources) {
  // The breadcrumb path of where an answer came from: one quiet
  // pill per source, pack › document, drawn under the message it
  // belongs to. Only a message that actually drew on a pack
  // carries any — once the conversation moves on, they stop.
  // The pills are returned BARE (no row wrapper of their own):
  // pillRow() owns the one row under a reply, so a Mesh pill and
  // source pills can share it — Build 8 (#88).
  if (!sources || !sources.length) return "";
  return sources.map(function (s) { return "<span class='srcpill'>" + esc(s.pack) + " <span class='srcsep' aria-hidden='true'>›</span> " + esc(s.document) + "</span>"; }).join("");
}
// Ghost-hint history for the chat composer: the CLI lines already
// run in this conversation, newest first — terminal blocks carry
// their command; operator lines count when they routed to the CLI.
function chatCliHistory() {
  var store = cliHistoryStore();
  if (store) { var out = [], seen = {}; for (var i = store.length - 1; i >= 0; i--) { var l = store[i]; if (l && !seen[l]) { seen[l] = 1; out.push(l); } } return out; }
  var out2 = [], seen2 = {};
  for (var j = messages.length - 1; j >= 0 && out2.length < 50; j--) { var m = messages[j]; var line = null; if (m.exec && !m.text) line = m.exec.command; else if (m.who === "YOU" && routeOfLine(m.text).kind === "cli") line = m.text; if (line && !seen2[line]) { seen2[line] = 1; out2.push(line); } }
  return out2;
}
function execContextText(x) {
  var out = String(x.output || "");
  var note = "";
  if (out.length > 1200) { note = "\n… [terminal output truncated for context — " + out.length + " characters in full]"; out = out.slice(0, 1200); }
  return "$ " + x.command + "\n" + out + note;
}
function liveChat(text, speak) {
  // Sliding history: only the last 20 messages ride to the model
  // (the slice above). Prefill is paid per token on CPU — an
  // ever-growing transcript would make every reply slower than the
  // last. The full conversation stays in this tab; only her view
  // of it slides. The bridge caps again on its side.
  // Reply presentation rule (the operator's call, stated plainly):
  // her answers are never typed out character by character. The
  // bridge request is non-streaming, the typing indicator covers
  // the whole wait, and when the reply is ready it renders complete,
  // all at once. Do not add a typewriter reveal here.
  // Terminal blocks ride as context: a block the operator produced
  // by running a command directly (CLI-first) becomes a system note
  // carrying the command and its output, and a block attached to
  // one of her replies folds into that reply's content — so her
  // next answer can reason over what the terminal showed (list a
  // folder, then ask her about a file in the listing) instead of
  // starting blind. Earlier builds filtered exec blocks out of
  // history entirely; that is exactly what CLI-first reverses.
  var hist = [];
  messages.slice(-20).forEach(function (x) {
    if (x.exec && !x.text) {
      hist.push({ role: "system", content: "Terminal output from earlier in this conversation — the operator ran this command directly in the chat:\n" + execContextText(x.exec) });
      return;
    }
    var content = x.who === "NAVI" ? stripCite(x.text) : String(x.text || "").replace(/^>\s+/, "");
    if (x.exec) content += "\n\n[terminal]\n" + execContextText(x.exec);
    // Build 6 (#224): a hollow history entry never rides — the
    // model server refuses the whole request over one empty
    // content (the call-killing 400). The bridge guards the
    // same line for every other caller; the page keeps its own
    // history clean at the source.
    if (!String(content).trim()) return;
    hist.push({ role: x.who === "YOU" ? "user" : "assistant", content: content });
  });
  var payload = { messages: hist.concat([{ role: "user", content: text }]), sudo: !!CONFIG.sudo, operator_name: CONFIG.operatorName || "", speak: !!speak, mode: chatMode, memory_mode: CONFIG.memoryMode || "autonomous", chatId: activeChatId };
  return fetch("/api/chat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) })
    .then(function (r) { return r.json().then(function (j) { if (!r.ok) throw new Error(j.error || "bridge error"); return j; }); })
    .then(function (j) { vaultLoadedAt = 0; if (j.exec) lastChatExec = { command: j.exec.command, output: j.exec.output, at: Date.now() }; return { text: stripCite(j.reply), exec: j.exec ? { command: j.exec.command, output: j.exec.output, exit_code: typeof j.exec.exit_code === "number" ? j.exec.exit_code : 0 } : null, sources: (j.rag && j.rag.length) ? j.rag : null, mode: j.mode || "model", operation: j.operation || null, memorySaves: j.memorySaves || null, memoryRefused: !!j.memoryRefused, mesh: j.mesh || null, caseFilings: j.caseFilings || null, refPulls: j.refPulls || null, caseLog: j.caseLog || null, caseIntake: j.caseIntake || null, chatId: activeChatId }; });
}
function isDhdChatIntent(text) {
  return /\bdhd\b/i.test(text || "") && /\b(turn on|turn off|switch on|switch off|enable|disable|start|stop|activate|set up|setup|open)\b/i.test(text || "") && !/^\s*(how|what|why|explain)\b/i.test(text || "");
}
function liveShell(cmd) {
  return fetch("/api/shell", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ command: cmd, sudo: !!CONFIG.sudo, sudo_password: "" }) })
    .then(function (r) { return r.json(); })
    .then(function (j) { return { command: j.command, output: j.output, exit_code: typeof j.exit_code === "number" ? j.exit_code : 0 }; });
}

// The chat bar's hint text, from the state she is in right now:
// on a call it speaks to the call; with the model disconnected —
// or the latch on CLI — it invites a command; otherwise it
// invites a question. One source for the render and for every
// live mode change (build .52, issue #12): the placeholder used
// to be picked only at render time, so flipping the CLI/Navi
// latch left the old mode's words sitting in the bar.
// Phone widths (build .67): at ≤800px the bar shares its row
// with the 44px clip/mic/call buttons, leaving ~165–195px of
// text room — the full teaching placeholders clipped mid-word
// in Alex's field photo even though every box measured fine.
// The narrow variants are short enough to sit fully visible
// with room to spare at 360px; desktop keeps the long forms.
function chatPlaceholder() {
  var narrow = false;
  try { narrow = window.innerWidth <= 800; } catch (e) {}
  if (call) return narrow ? "Type what you'd say…" : "You're on a call — type what you'd say…";
  if (chatMode === "cli" && CONFIG.backend === "live") return narrow ? "Shell session…" : "Shell session — type a line, Enter sends it";
  if (llama.status !== "running" || chatMode === "cli") return narrow ? "Run a command…" : "Run a command — try: ls";
  return narrow ? "Ask Navi…" : "Ask Navi — try: show me the contents of /home/name/";
}

/* ----------------------------- autocomplete ----------------------------- */
// One completion system with two completers: the dropdown completes
// the command word from the system's commands; the ghost —
// zsh-autosuggestions style — completes the whole line from command
// history. They never fight: while the dropdown is open it owns
// Tab/Enter/arrows (and swallows the keystroke whole, so accepting
// a suggestion never also submits the line); the ghost shows only
// when the dropdown is closed, and Tab or Right Arrow accepts it.
// In rich mode the input also wears the highlight layer — the typed
// line, colored as it's typed when it routes to the CLI — and, when
// a hint element is supplied, drives the route hint under the
// composer, whose button flips the line's route by rewriting its
// marker prefix.
function attachAutocomplete(input, opts) {
  if (!input || input.dataset.acReady) return;
  input.dataset.acReady = "1";
  opts = opts || {};
  var wrap = input.parentElement;
  wrap.classList.add("ac-wrap");
  var box = document.createElement("div");
  box.className = "ac"; box.style.display = "none"; box.setAttribute("role", "listbox"); box.setAttribute("aria-label", "Command suggestions");
  wrap.appendChild(box);
  var searchEl = document.createElement("div");
  searchEl.className = "acsearch"; searchEl.style.display = "none";
  wrap.appendChild(searchEl);
  var layer = null, ghostLine = "", ghostOffFor = "";
  if (opts.rich) {
    layer = document.createElement("div");
    layer.className = "hlayer";
    layer.setAttribute("aria-hidden", "true");
    wrap.insertBefore(layer, input);
    input.classList.add("rich");
  }
  var items = [], active = 0, tokCtx = null;
  var recall = { active: false, idx: -1, stash: "" };
  var search = { active: false, q: "", idx: 0, stash: "", matches: [] };
  var killBuf = "";
  var armedPreview = null;
  var compSeq = 0;
  function lineRoute() { return routeOfLine(input.value, chatMode); }
  function histNote(msg) {
    if (!opts.hintEl) return;
    var n = opts.hintEl.querySelector("#hist-note");
    if (!n) { n = document.createElement("span"); n.id = "hist-note"; n.style.color = "var(--am)"; opts.hintEl.appendChild(n); }
    n.textContent = msg || "";
  }
  function latchMode(m) {
    chatMode = m;
    persistChatMode();
    chatModeToggled(m);
    input.placeholder = chatPlaceholder();
    var v = input.value;
    if (m === "cli" && (v.trim() === ">" || v.indexOf("> ") === 0)) input.value = v.replace(/^>\s*/, "");
    else if (m === "navi" && v.charAt(0) === "$") input.value = v.replace(/^\$\s*/, "");
  }
  function close() { box.style.display = "none"; items = []; tokCtx = null; input.dataset.acOpen = ""; }
  function replaceToken(c) {
    var v = input.value, caret = input.selectionStart == null ? v.length : input.selectionStart;
    if (!tokCtx) { input.value = c.name + " "; return false; }
    var startT = tokCtx.tokStart;
    var endT = caret; while (endT < v.length && !/\s/.test(v.charAt(endT))) endT++;
    var rep = c.replacement || c.name;
    input.value = v.slice(0, startT) + rep + v.slice(endT);
    var pos = startT + rep.length;
    try { input.setSelectionRange(pos, pos); } catch (e) {}
    return rep.charAt(rep.length - 1) === "/";
  }
  function accept(i) {
    var c = items[i]; if (!c) return;
    var reopen = replaceToken(c);
    close(); input.focus();
    if (reopen) refresh(); else refresh();
  }
  function paint() {
    box.innerHTML = items.map(function (c, i) { return "<div class='ac-it" + (i === active ? " on" : "") + "' role='option' data-i='" + i + "'><b>" + esc(c.name) + "</b><span>" + esc(c.description) + "</span><em>" + esc(c.source === "default" ? "built-in" : c.source === "installed" ? "installed" : String(c.source || "")) + "</em></div>"; }).join("");
    box.querySelectorAll(".ac-it").forEach(function (el) { el.addEventListener("mousedown", function (e) { e.preventDefault(); accept(Number(el.getAttribute("data-i"))); }); });
  }
  function findGhost() {
    if (!opts.rich || !opts.history) return "";
    var v = input.value;
    if (!v || ghostOffFor === v) return "";
    if (input.selectionStart !== v.length) return "";
    if (lineRoute().kind !== "cli") return "";
    var hist = opts.history() || [];
    for (var i = 0; i < hist.length; i++) { var h = hist[i]; if (h.length > v.length && h.toLowerCase().indexOf(v.toLowerCase()) === 0) return h; }
    return "";
  }
  function paintLayer() {
    if (layer) {
      var v = input.value;
      var html = (lineRoute().kind === "cli" || looksLikeCommandLine(v)) ? cmdTokensHtml(v) : esc(v);
      if (ghostLine) html += "<span class='ghost'>" + esc(ghostLine.slice(v.length)) + "</span>";
      layer.innerHTML = html;
      layer.scrollLeft = input.scrollLeft;
    }
    // The bar dresses for the latch (build .66): CLI latched, its
    // edge takes the terminal green; Navi latched, it lets go.
    wrap.classList.toggle("cli-mode", chatMode === "cli");
    if (opts.hintEl) {
      var btn = opts.hintEl.querySelector("button.pill");
      if (btn) {
        var r = lineRoute();
        btn.textContent = r.kind === "cli" ? (chatMode === "cli" && CONFIG.backend === "live" ? "⏎ CLI — types into the shell session" : "⏎ CLI — runs as a system command") : "⏎ Navi — goes to Navi";
        btn.className = r.kind === "cli" ? "pill cli" : "pill";
      }
      opts.hintEl.querySelectorAll("button[data-mode]").forEach(function (b) {
        var m = b.getAttribute("data-mode");
        var on = chatMode === m;
        b.setAttribute("aria-pressed", on ? "true" : "false");
        b.className = on ? (m === "cli" ? "on-cli" : "on-navi") : "";
      });
    }
  }
  function searchMatches() {
    var hist = chatCliHistory();
    search.matches = search.q ? hist.filter(function (h) { return h.toLowerCase().indexOf(search.q.toLowerCase()) !== -1; }) : hist.slice();
    if (search.idx >= search.matches.length) search.idx = Math.max(0, search.matches.length - 1);
  }
  function paintSearch() {
    if (!search.active) { searchEl.style.display = "none"; return; }
    var cur = search.matches[search.idx] || null;
    searchEl.innerHTML = "<b>reverse search</b> <span class='mono'>" + esc(search.q) + "</span> — " + (cur ? "<span class='mono'>" + esc(cur) + "</span>" : "<span>no match</span>");
    searchEl.style.display = "block";
  }
  function endSearch(restore) {
    if (restore) input.value = search.stash;
    search.active = false; paintSearch(); refresh();
  }
  function refresh() {
    var v = input.value;
    var caret = input.selectionStart == null ? v.length : input.selectionStart;
    var open = false; tokCtx = null;
    if (v && v.indexOf("./") === 0 && v.indexOf(" ") === -1 && CONFIG.backend === "live" && wfData && chatMode !== "cli") {
      // ./ completes chip names (0.1.9e, issue #97): the shelf,
      // offered as you type the invocation itself.
      var dtok = v.slice(2).toLowerCase();
      items = wfData.chips.filter(function (c) { return !c.unreadable; }).filter(function (c) { var slug = String(c.file || "").replace(/.png$/i, ""); return slug.toLowerCase().indexOf(dtok) === 0 || slug.toLowerCase().replace(/^navi-/, "").indexOf(dtok) === 0; }).map(function (c) { var slug = String(c.file || "").replace(/.png$/i, ""); return { name: "./" + slug, description: String(c.name || slug) + " — chip", source: "chip" }; }).slice(0, 8);
      if (items.length) { active = 0; paint(); box.style.display = "block"; input.dataset.acOpen = "1"; open = true; }
    }
    if (!open && v && v.indexOf(" ") === -1 && caret <= v.indexOf(" ") + 1) {
      var t = v.toLowerCase();
      items = getCommands().filter(function (c) { return c.name.indexOf(t) === 0 && c.name !== t; }).slice(0, 8);
      if (items.length) { active = 0; paint(); box.style.display = "block"; input.dataset.acOpen = "1"; open = true; }
    }
    if (!open && v && v.indexOf(" ") !== -1) {
      var ctx = tokenItemsForLine(v, caret);
      if (ctx) {
        tokCtx = ctx;
        if (ctx.isPathy) {
          var seq = ++compSeq; var captured = v;
          var dirKey = ctx.tok.slice(0, ctx.tok.lastIndexOf("/") + 1);
          var done = function (entries) {
            if (seq !== compSeq || input.value !== captured) return;
            items = entries.map(function (e2) { return { name: e2.name, description: e2.description, source: e2.isDir ? "folder" : "file", replacement: e2.replacement }; });
            if (items.length) { active = 0; paint(); box.style.display = "block"; input.dataset.acOpen = "1"; } else close();
            ghostLine = ""; paintLayer();
          };
          if (pathCompleteCache[dirKey]) done(pathCompleteCache[dirKey]);
          else if (CONFIG.backend === "live") fetch("/api/pathcomplete?prefix=" + encodeURIComponent(ctx.tok)).then(function (r) { return r.json(); }).then(function (j) { pathCompleteCache[dirKey] = j.entries || []; done(j.entries || []); }).catch(function () { done([]); });
          else done(mockPathComplete(ctx.tok));
          items = ctx.items;
          if (items.length) { active = 0; paint(); box.style.display = "block"; input.dataset.acOpen = "1"; open = true; }
        } else if (ctx.items.length) {
          items = ctx.items; active = 0; paint(); box.style.display = "block"; input.dataset.acOpen = "1"; open = true;
        }
      }
    }
    if (!open && box.style.display !== "block") close();
    else if (!open) close();
    ghostLine = open ? "" : findGhost();
    paintLayer();
  }
  if (opts.hintEl) {
    var pillBtn = opts.hintEl.querySelector("button.pill");
    if (pillBtn) pillBtn.onclick = function () { latchMode(lineRoute().kind === "cli" ? "navi" : "cli"); input.focus(); refresh(); };
    opts.hintEl.querySelectorAll("button[data-mode]").forEach(function (b) { b.onclick = function () { latchMode(b.getAttribute("data-mode") || "navi"); input.focus(); refresh(); }; });
  }
  input.addEventListener("input", function () {
    recall.active = false;
    if (armedPreview !== null && input.value !== armedPreview) { armedPreview = null; histNote(""); }
    refresh();
  });
  input.addEventListener("focus", refresh);
  input.addEventListener("scroll", function () { if (layer) layer.scrollLeft = input.scrollLeft; });
  input.addEventListener("blur", function () { setTimeout(close, 150); });
  input.addEventListener("keydown", function (e) {
    var ctrl = e.ctrlKey || e.metaKey;
    if (ctrl && (e.key === "r" || e.key === "R")) {
      e.preventDefault(); e.stopImmediatePropagation();
      if (!search.active) { search.active = true; search.q = ""; search.idx = 0; search.stash = input.value; searchMatches(); }
      else if (search.matches.length) search.idx = Math.min(search.idx + 1, search.matches.length - 1);
      paintSearch(); return;
    }
    if (search.active) {
      if (e.key === "Escape") { e.preventDefault(); e.stopImmediatePropagation(); endSearch(true); return; }
      if (e.key === "Enter" || e.key === "Tab") { e.preventDefault(); e.stopImmediatePropagation(); var cur = search.matches[search.idx]; var keep = cur || search.stash; search.active = false; paintSearch(); if (cur) input.value = cur; refresh(); return; }
      if (e.key === "ArrowDown") { e.preventDefault(); if (search.matches.length) search.idx = Math.min(search.idx + 1, search.matches.length - 1); paintSearch(); return; }
      if (e.key === "ArrowUp") { e.preventDefault(); if (search.matches.length) search.idx = Math.max(search.idx - 1, 0); paintSearch(); return; }
      if (e.key === "Backspace") { e.preventDefault(); search.q = search.q.slice(0, -1); search.idx = 0; searchMatches(); paintSearch(); return; }
      if (e.key.length === 1 && !ctrl) { e.preventDefault(); search.q += e.key; search.idx = 0; searchMatches(); paintSearch(); return; }
      return;
    }
    if (ctrl && opts.rich) {
      var v0 = input.value, c0 = input.selectionStart == null ? v0.length : input.selectionStart;
      var setV = function (nv, pos) { input.value = nv; try { input.setSelectionRange(pos, pos); } catch (err) {} recall.active = false; refresh(); };
      if (e.key === "a" || e.key === "A") { e.preventDefault(); try { input.setSelectionRange(0, 0); } catch (err2) {} paintLayer(); return; }
      if (e.key === "e" || e.key === "E") { e.preventDefault(); try { input.setSelectionRange(v0.length, v0.length); } catch (err3) {} paintLayer(); return; }
      if (e.key === "u" || e.key === "U") { e.preventDefault(); if (c0 > 0) { killBuf = v0.slice(0, c0); setV(v0.slice(c0), 0); } return; }
      if (e.key === "k" || e.key === "K") { e.preventDefault(); if (c0 < v0.length) { killBuf = v0.slice(c0); setV(v0.slice(0, c0), c0); } return; }
      if (e.key === "w" || e.key === "W") { e.preventDefault(); var st = c0; while (st > 0 && /\s/.test(v0.charAt(st - 1))) st--; while (st > 0 && !/\s/.test(v0.charAt(st - 1))) st--; if (st < c0) { killBuf = v0.slice(st, c0); setV(v0.slice(0, st) + v0.slice(c0), st); } return; }
      if (e.key === "y" || e.key === "Y") { e.preventDefault(); if (killBuf) setV(v0.slice(0, c0) + killBuf + v0.slice(c0), c0 + killBuf.length); return; }
    }
    if (box.style.display !== "none") {
      if (e.key === "ArrowDown") { e.preventDefault(); active = (active + 1) % items.length; paint(); }
      else if (e.key === "ArrowUp") { e.preventDefault(); active = (active - 1 + items.length) % items.length; paint(); }
      else if (e.key === "Tab" || e.key === "Enter") { e.preventDefault(); e.stopImmediatePropagation(); accept(active); }
      else if (e.key === "Escape") { close(); }
      return;
    }
    if ((e.key === "ArrowUp" || e.key === "ArrowDown") && opts.rich) {
      var hist2 = chatCliHistory();
      var v2 = input.value;
      var engages = recall.active || v2 === "" || lineRoute().kind === "cli" || hist2.indexOf(v2) !== -1 || (llama && llama.status !== "running") || chatMode === "cli";
      if (engages && hist2.length) {
        e.preventDefault();
        if (!recall.active) { recall.active = true; recall.stash = v2; recall.idx = e.key === "ArrowUp" ? 0 : -1; }
        else if (e.key === "ArrowUp") recall.idx = Math.min(recall.idx + 1, hist2.length - 1);
        else recall.idx = recall.idx - 1;
        if (recall.idx < 0) { recall.active = false; input.value = recall.stash; }
        else input.value = hist2[recall.idx];
        refresh(); return;
      }
    }
    if (e.key === "Enter" && opts.rich && armedPreview === null) {
      var rv = input.value;
      var rr = routeOfLine(rv, chatMode);
      var isCliLine = rr.kind === "cli" || (llama && llama.status !== "running" && !/^\s*\/(chip|rag)\b/i.test(rv));
      if (isCliLine && rv.indexOf("!") !== -1) {
        var ex = expandHistoryRefs(rv, cliHistoryStore() || []);
        if (ex.error) { e.preventDefault(); e.stopImmediatePropagation(); histNote("History expansion: " + ex.error + " — nothing was sent."); return; }
        if (ex.changed) { e.preventDefault(); e.stopImmediatePropagation(); input.value = ex.line; armedPreview = ex.line; histNote("History expansion — check it, then press Enter again to run: " + ex.line); refresh(); return; }
      }
    }
    if (e.key === "Enter" && armedPreview !== null && input.value === armedPreview) { armedPreview = null; histNote(""); }
    if (ghostLine && (e.key === "Tab" || e.key === "ArrowRight")) { e.preventDefault(); input.value = ghostLine; ghostLine = ""; refresh(); return; }
    if (e.key === "Escape" && ghostLine) { ghostOffFor = input.value; ghostLine = ""; paintLayer(); }
  });
  paintLayer();
}

/* -------------------------------- render -------------------------------- */
applyTheme();
applyMotion();
applyLayout();
applyUiScale();
var appEl = document.getElementById("app"), navEl = document.getElementById("nav"), mainEl = document.getElementById("view-root"), statusEl = document.getElementById("status");
var toggleBtn = document.createElement("button");
toggleBtn.id = "nav-toggle";
toggleBtn.setAttribute("aria-expanded", navClosed ? "false" : "true");
toggleBtn.setAttribute("aria-label", "Collapse or expand the side menu");
toggleBtn.textContent = navClosed ? (window.innerWidth <= 800 ? "☰" : "»") : (window.innerWidth <= 800 ? "☰" : "« Menu");
toggleBtn.onclick = function () {
  navClosed = !navClosed;
  // Opening the menu puts the dock away (build .64): on a phone
  // only one overlay is ever up.
  if (!navClosed) dockOpen = false;
  try { localStorage.setItem("navi.navClosed", navClosed ? "1" : "0"); } catch (e) {}
  render();
};
navEl.appendChild(toggleBtn);
// The destinations and the version footer ride in one container:
// display:contents on desktop (the nav's flex column is unchanged),
// the full-width overlay panel on small screens.
var navMenu = document.createElement("div");
navMenu.id = "nav-menu";
navEl.appendChild(navMenu);
function goView(name) {
  view = name;
  if (window.innerWidth <= 800) { navClosed = true; dockOpen = false; }
  render();
  if (name === "Settings") { if (updater.state === "idle") checkForUpdates(); fetchUpdateResult(updater.state === "applying"); }
  // 0.2.0e Build 10 (#195): the federation state is fetched
  // ONCE per view entry — here, at the door, never from inside
  // render. An answer that sets nothing is held in
  // federationError and painted once by the view itself.
  if (name === "Federation" && CONFIG.backend === "live" && !federationState && !federationFetching) federationFetchState();
}
function navBtn(id, label, iconName, targetView) {
  var b = document.createElement("button");
  b.type = "button";
  b.id = id;
  b.setAttribute("aria-label", label);
  b.innerHTML = "<span class='ic'>" + navIcon(iconName || label) + "</span><span class='lbl'>" + label + "</span><span class='nav-tip' aria-hidden='true'>" + label + "</span>";
  b.onclick = function () {
    // Build 4 (#213/#216): the two entries onto the Wire
    // surface land on their own ground, so the surface showing
    // and the rail's single selected entry always agree \u2014
    // Home lands on the board; The Wire lands on its head
    // family when the board is what's showing.
    if (id === "nav-Home") setIntelTab("home");
    else if (id === "nav-child-apps-The Wire" && intelTab === "home") setIntelTab("news");
    goView(targetView);
  };
  return b;
}
function navGroupHeader(gid) {
  var g = NAV_GROUPS[gid];
  var b = document.createElement("button");
  b.type = "button";
  b.id = "nav-group-" + gid;
  b.className = "nav-group-head";
  b.setAttribute("aria-expanded", navGroupsOpen[gid] ? "true" : "false");
  b.setAttribute("aria-controls", "nav-children-" + gid);
  b.setAttribute("aria-label", g.label + " — expand or collapse the group");
  b.innerHTML = "<span class='ic'>" + navIcon(g.label) + "</span><span class='lbl'>" + g.label + "</span><span class='nav-chev' aria-hidden='true'>▾</span><span class='nav-tip' aria-hidden='true'>" + g.label + "</span>";
  b.onclick = function () {
    // On the collapsed rail the header is a door, not a toggle:
    // it opens the rail so the children can be reached.
    if (document.getElementById("app").classList.contains("closed") && window.innerWidth > 800) { navClosed = false; try { localStorage.setItem("navi.navClosed", "0"); } catch (e) {} render(); return; }
    navGroupsOpen[gid] = !navGroupsOpen[gid];
    syncNav();
  };
  return b;
}
function navGroupWrap(gid) {
  var wrap = document.createElement("div");
  wrap.className = "nav-children" + (navGroupsOpen[gid] ? "" : " closed");
  wrap.id = "nav-children-" + gid;
  // Build 6 (#105): the buttons ride in ONE inner track — the
  // grid animates a single templated row around it (the Docs
  // topics' shape). Build 4 appended the buttons as direct
  // grid children, so only the first row was templated and the
  // rest overflowed the track onto whatever followed.
  var inner = document.createElement("div");
  inner.className = "nav-children-inner";
  NAV_GROUPS[gid].children.forEach(function (pair, i) {
    // Build 13 (#149, bake find): the icon resolves by the
    // VIEW KEY (pair[1]), not the label — "Case Files" is the
    // one label that differs from its key, and by label its
    // rail icon had rendered as an empty svg since Build 12.
    var b = navBtn("nav-child-" + gid + "-" + pair[0], pair[0], pair[1], pair[1]);
    b.classList.add("nav-child");
    b.style.transitionDelay = (i * 55) + "ms";
    if (pair[1] === "Sync") {
      // Build 3 (#176): Sync is not a view — it is the Quick
      // Sync control. Tapping it opens no page; it pops the sync
      // card and runs the round, so the nav button's view change
      // is replaced outright.
      b.onclick = function () { openSyncCard(); };
    }
    inner.appendChild(b);
  });
  wrap.appendChild(inner);
  return wrap;
}
// The menu (0.1.9e Build 4; Build 6, #106; Build 14, #160):
// Chat pinned; a break; then the groups in the settled order —
// Navi, Casework, Network — each a section of
// header, children, and its OWN break as the section's last
// element, so the divider travels with the group and sits
// directly beneath its last visible row in every state (a
// standalone sibling used to be stranded a full group-height
// below a collapsed header, because the collapsed children
// stayed in the flow); and at the foot, low by the version
// line, the Utilities drop-up over the version line —
// the group seated low opens UPWARD, its children stacked
// above its header (#160 settles both the order and that
// direction; group contents are unchanged).
// Build 3 (#210): Home \u2014 the fused Wire surface (the Center
// and the Dashboard, one page) \u2014 sits at the very top of the
// rail and is what the app opens to. Build 4 (#216): it shares
// the surface with Apps \u2192 The Wire, but selection is singular
// \u2014 the board lights Home alone, a family lights The Wire
// alone (see syncNav).
navMenu.appendChild(navBtn("nav-Home", "Home", "Home", "ThreatIntel"));
navMenu.appendChild(navBtn("nav-Chat", "Chat", "Chat", "Chat"));
var navBrk1 = document.createElement("div"); navBrk1.className = "nav-break"; navBrk1.setAttribute("aria-hidden", "true"); navMenu.appendChild(navBrk1);
var navSys = document.createElement("div"); navSys.className = "nav-group-sec"; navSys.id = "nav-sec-network";
navSys.appendChild(navGroupHeader("network"));
navSys.appendChild(navGroupWrap("network"));
var navBrk2 = document.createElement("div"); navBrk2.className = "nav-break"; navBrk2.setAttribute("aria-hidden", "true"); navSys.appendChild(navBrk2);
var navNavi = document.createElement("div"); navNavi.className = "nav-group-sec"; navNavi.id = "nav-sec-navi";
navNavi.appendChild(navGroupHeader("navi"));
navNavi.appendChild(navGroupWrap("navi"));
var navBrk3 = document.createElement("div"); navBrk3.className = "nav-break"; navBrk3.setAttribute("aria-hidden", "true"); navNavi.appendChild(navBrk3);
navMenu.appendChild(navNavi);
var navApps = document.createElement("div"); navApps.className = "nav-group-sec"; navApps.id = "nav-sec-apps";
navApps.appendChild(navGroupHeader("apps"));
navApps.appendChild(navGroupWrap("apps"));
var navBrk4 = document.createElement("div"); navBrk4.className = "nav-break"; navBrk4.setAttribute("aria-hidden", "true"); navApps.appendChild(navBrk4);
navMenu.appendChild(navApps);
navMenu.appendChild(navSys);
var navSiem = document.createElement("div"); navSiem.className = "nav-group-sec"; navSiem.id = "nav-sec-siem";
navSiem.appendChild(navGroupHeader("siem"));
navSiem.appendChild(navGroupWrap("siem"));
var navBrk5 = document.createElement("div"); navBrk5.className = "nav-break"; navBrk5.setAttribute("aria-hidden", "true"); navSiem.appendChild(navBrk5);
navMenu.appendChild(navSiem);
var navUtil = document.createElement("div"); navUtil.className = "nav-util"; navUtil.id = "nav-util";
navUtil.appendChild(navGroupWrap("utilities"));
navUtil.appendChild(navGroupHeader("utilities"));
navMenu.appendChild(navUtil);
// syncNav: the active sweep + group states, applied to the static
// menu DOM on every render WITHOUT rebuilding it — a rebuilt menu
// would restart the groups' open/close transitions on every status
// poll; this sweep restarts nothing. The DHD view lights its own
// "DHD" child (the Node child is a shortcut entry to the same
// view and never carries the active state — Build 4's honest
// choice, noted in the build report).
var NAV_ACTIVE_IDS = { Chat: "nav-Chat", Chips: "nav-child-navi-Chips", ChipDeck: "nav-child-navi-ChipDeck", RAG: "nav-child-navi-RAG", CaseFiles: "nav-child-apps-Case Files", ThreatIntel: "nav-child-apps-The Wire", Vault: "nav-child-apps-Vault", GitHub: "nav-child-utilities-GitHub", Docs: "nav-child-utilities-Docs", Settings: "nav-child-utilities-Settings", Guardian: "nav-child-utilities-Guardian", Mesh: "nav-child-network-Mesh", DHD: "nav-child-network-DHD", Federation: "nav-child-network-Federation", FediChat: "nav-child-network-FediChat", Dossier: "nav-child-apps-The Dossier", Seraph: "nav-child-siem-Seraph" };
function syncNav() {
  var want = NAV_ACTIVE_IDS[view] || "";
  // Build 4 (#216): selection is SINGULAR. Home and The Wire
  // are two entries onto one surface, but only one ever
  // lights: the board lights Home; any family view lights
  // The Wire (the families' entry); the CTFBoard view is the
  // CTF family, so it lights The Wire too. (Build 3 lit both
  // entries whenever the surface showed \u2014 the fusion never
  // disentangled the two selected states.)
  if (view === "ThreatIntel") want = (intelTab === "home") ? "nav-Home" : "nav-child-apps-The Wire";
  else if (view === "CTFBoard") want = "nav-child-apps-The Wire";
  ["nav-Home", "nav-Chat", "nav-child-network-Federation", "nav-child-network-FediChat", "nav-child-network-Mesh", "nav-child-network-DHD", "nav-child-navi-Chips", "nav-child-navi-ChipDeck", "nav-child-navi-RAG", "nav-child-apps-Case Files", "nav-child-apps-The Wire", "nav-child-apps-Vault", "nav-child-apps-The Dossier", "nav-child-siem-Seraph", "nav-child-utilities-Settings", "nav-child-utilities-GitHub", "nav-child-utilities-Docs", "nav-child-utilities-Guardian"].forEach(function (id) {
    var el = document.getElementById(id);
    if (el) el.className = el.className.replace(/\bon\b/g, "").replace(/\s+/g, " ").trim();
    if (el && id === want) el.className = (el.className ? el.className + " " : "") + "on";
  });
  // v0.4.10e (#249): the rail head's Home tile carries the
  // same active state as the labeled Home row — one selection,
  // painted on both seats of Home.
  var railHomeEl = document.getElementById("rail-home");
  if (railHomeEl) railHomeEl.classList.toggle("on", want === "nav-Home");
  ["network", "navi", "apps", "siem", "utilities"].forEach(function (gid) {
    var wrap = document.getElementById("nav-children-" + gid);
    if (wrap) {
      var closed = !navGroupsOpen[gid];
      if (wrap.classList.contains("closed") !== closed) wrap.classList.toggle("closed", closed);
      wrap.setAttribute("aria-hidden", closed ? "true" : "false");
    }
    var head = document.getElementById("nav-group-" + gid);
    if (head) head.setAttribute("aria-expanded", navGroupsOpen[gid] ? "true" : "false");
  });
}
// The dock toggle (build .64): the opposing end of the phone top
// bar from the hamburger. It opens the whole status column as a
// full-width overlay; the small-screen CSS shows it, the desktop
// CSS keeps it display:none — the desktop frame never grows one.
var dockBtn = document.createElement("button");
dockBtn.id = "dock-toggle";
dockBtn.type = "button";
dockBtn.innerHTML = statusPanelSvg();
dockBtn.setAttribute("aria-label", "Open the status dock");
dockBtn.setAttribute("aria-expanded", "false");
dockBtn.onclick = function () {
  dockOpen = !dockOpen;
  if (dockOpen) { navClosed = true; statusPaneTab = "status"; }
  render();
};
navEl.appendChild(dockBtn);
// The notifications bell (v0.4.7e, #242; moved v0.4.8e, #247;
// paired v0.4.10e, #249): created here, in the toggleBtn/dockBtn
// idiom, and seated in #rail-head BEFORE #nav-menu — at the head
// of the rail beside Home, on the desktop canvas and through
// DHD alike. As first shipped it was appended last and
// rendered at the foot of the rail, under the version line;
// the operator ruled the bell belongs at the head of the
// column. It still never hides inside the collapsible
// #nav-menu, and on the phone it rides the top bar in its #242
// slot (the ≤800px order guard in the stylesheet). Its badge
// paints from notifState via paintNotifBadge(); the button
// binds once, here.
var notifBtn = document.createElement("button");
notifBtn.id = "notif-bell";
notifBtn.type = "button";
notifBtn.innerHTML = "<span class='ic'><svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M6 8a6 6 0 0 1 12 0c0 7 3 9 3 9H3s3-2 3-9'/><path d='M10.3 21a1.94 1.94 0 0 0 3.4 0'/></svg></span><span id='notif-badge' style='display:none'>0</span>";
notifBtn.setAttribute("aria-label", "Notifications");
notifBtn.onclick = function () { openNotifPanel(); };
// v0.4.10e (#249): the bell gains a partner — the head of the
// rail is an icon pair now. #rail-head seats where the bell
// sat alone (before #nav-menu, a direct child of the nav) and
// holds the bell and an icon-only Home tile side by side. The
// tile does exactly what the labeled Home row does — same
// tab, same view — and syncNav mirrors Home's active state
// onto it, so the filled tile IS Home's state on the wide
// canvas, where the stylesheet retires the labeled row. The
// words live in the tooltip and the aria-labels. Collapsed,
// the pair stacks bell over Home (stylesheet); on the phone
// the head dissolves and nothing moves.
var railHead = document.createElement("div");
railHead.id = "rail-head";
var railHome = document.createElement("button");
railHome.id = "rail-home";
railHome.type = "button";
railHome.innerHTML = "<span class='ic'>" + navIcon("Home") + "</span><span class='nav-tip' aria-hidden='true'>Home</span>";
railHome.setAttribute("aria-label", "Home");
railHome.title = "Home";
railHome.onclick = function () { setIntelTab("home"); goView("ThreatIntel"); };
notifBtn.title = "Notifications";
railHead.appendChild(notifBtn);
railHead.appendChild(railHome);
navEl.insertBefore(railHead, navMenu);
// Tapping back into the content dismisses whichever overlay is up
// (build .64) — reachable whenever an overlay leaves content
// exposed, and the same gesture on the bar's own toggles.
mainEl.addEventListener("click", function (e) {
  // The chat header's own status toggle (issue #115) lives inside
  // main: its click OPENS the dock, so the dismiss gesture must
  // not treat it as a tap on the content.
  if (window.innerWidth <= 800 && (dockOpen || !navClosed)) {
    dockOpen = false;
    navClosed = true;
    render();
  }
});
document.addEventListener("keydown", function (e) {
  if (e.key === "Escape" && window.innerWidth <= 800 && (dockOpen || !navClosed)) {
    dockOpen = false;
    navClosed = true;
    render();
  }
});
// The version at the foot of the menu, under its divider: baked
// from the package's VERSION file at build time, then kept current
// by the bridge's /api/status (which reads that same file) — one
// source of truth for the menu, the status column, and the update
// checker alike.
function verTag(st) { return st === "edge" || st === "alpha" ? "e" : st === "beta" ? "b" : ""; }
var navVer = document.createElement("div");
navVer.id = "nav-version";
navVer.className = "mono";
navVer.setAttribute("aria-label", "Apex version " + CONFIG.version);
navVer.innerHTML = "<span class='nv-num' id='nav-version-num'><span class='nv-pre'>v</span><span class='nv-val' id='nav-version-val'>" + esc(CONFIG.version) + verTag(CONFIG.versionStatus) + "</span></span>";
navVer.classList.add("st-" + (CONFIG.versionStatus || "stable"));
navMenu.appendChild(navVer);
/* ------------------------- Her living details ------------------------- */
// The tagline under the wordmark is one static line — "Powered by
// Navi" — baked into the dashboard HTML; it never cycles. Her
// presence line in the bar is the living one: its timer
// re-randomizes after every change and keeps the current text in
// module state, repainting whatever element is on the page
// — the frame re-renders constantly, so the cycler never rides a
// render. (v0.4.9e: the status pane no longer renders its own
// copy of the line — the bar is its one home.)
var PRESENCE_LINES = ["Navi is here.", "Navi is listening to the network breathe.", "Navi is reading Sigma rules like bedtime stories.", "Navi is in a staring contest with a firewall.", "Navi is poking a honeypot to see if it pokes back.", "Navi is counting packets until she falls asleep.", "Navi is reorganizing the chip shelf. Again.", "Navi is haunting Rockport's west end.", "Navi is arguing with a CAPTCHA. And winning.", "Navi is pretending port 22 doesn't exist.", "Navi is sharpening her scans.", "Navi is running on caffeine and spite.", "Navi is watching a log file grow. Riveting.", "Navi is somewhere between the packets.", "Navi is defragmenting her opinions.", "Navi is here. Allegedly.", "Navi is guarding servers with more firewalls than the devil's bedroom.", "Navi is chromed up and nowhere near flatlined.", "Navi is running hotter than a Night City summer.", "Navi is sandevistan-fast, consequences pending.", "Navi has seen things you people wouldn't believe. Mostly log files.", "Navi is keeping moments from being lost in time — they're in the case file.", "Navi changed the sky over the port. It was a dead channel.", "Navi is wintermute-proof. She checked.", "Navi is counting sheep in hexadecimal.", "Navi is locally hosted and globally unbothered.", "Navi pleads the fifth packet.", "Navi is holding the door like it's a zero-day.", "Navi doesn't chase packets. Packets confess.", "Navi is powered by spite, espresso, and other people's GPUs.", "Navi is somewhere between the console and the confessional.", "Navi is off the books, on the record, under the streetlights.", "Navi is in her console-cowgirl era. Deck's hot. Coffee's hotter.", "Navi is haunting the arcology's service tunnels.", "Navi is defying the Turing police, politely.", "Navi is one bad packet from a monologue.", "Avoid the ants, they gave Jack a tummy ache!", "Shhh... I'm listening to Darknet Diaries."];
// The rare one: on any given cycle it has roughly a 1-in-50 chance
// of being picked instead of the next regular line — an Easter egg
// that almost never shows. It stands as-is, not in the "Navi is…"
// shape, and it replaces that cycle's regular advance.
var PRESENCE_RARE = "Saving Ciel's ass... Again.";
// The startup pick is drawn at random from the whole pool
// (v0.4.9e) — she opens on any of her lines, not always the
// first — and every advance draws the same way.
var presenceIdx = Math.floor(Math.random() * PRESENCE_LINES.length);
var presenceText = PRESENCE_LINES[presenceIdx];
// v0.4.9e (#248, leg 5): lines the BRIDGE composes from
// safe, named material only — her version, a waiting
// update, the bell's kinds and counts, Mesh and federation
// state — ride in /api/status (presenceExtra) and join the
// same draw. Template-composed by the bridge, never
// model-written, never from case contents, chat contents,
// or operator-private detail. The canned pool is the
// floor: with no extras standing, the draw is the pool.
var presenceExtra = [];
function presencePool() {
  return presenceExtra.length ? PRESENCE_LINES.concat(presenceExtra) : PRESENCE_LINES;
}
// An advance draws uniformly from the pool with the line on
// screen excluded, so the status visibly changes every time
// and never repeats itself back-to-back (v0.4.9e). The rare
// one is never in the draw: it stands apart, as it always has.
function drawPresenceIdx() {
  var pool = presencePool();
  if (presenceIdx >= pool.length) presenceIdx = 0;
  var n = Math.floor(Math.random() * (pool.length - 1));
  return n >= presenceIdx ? n + 1 : n;
}
function cyclePresence() {
  if (Math.random() < 1 / 50) {
    presenceText = PRESENCE_RARE;
  } else {
    presenceIdx = drawPresenceIdx();
    presenceText = presencePool()[presenceIdx];
  }
  var el = document.getElementById("presence-line");
  if (el) el.textContent = presenceText;
  var cel = document.getElementById("chat-presence-line");
  if (cel && !call && !naviThinking && CONFIG.backend === "live" && llama.status === "running") cel.textContent = presenceText;
  setTimeout(cyclePresence, 120000 + Math.random() * 120000);


}
setTimeout(cyclePresence, 120000 + Math.random() * 120000);
// Her presence, computed once (Build 6, #115): her presence
// bar (Build 8 — frame chrome over every view) reads from
// this, and only this —
// and in the state where she is simply here, its text IS the
// living line (presenceText above) — the bar is the line's one
// rendered home since v0.4.9e, when the status pane's own copy
// was retired; this adds the state variants: in a call, away,
// simulated. (v0.4.1e, #228: there is no typing variant — the
// top banner no longer carries a "Navi is typing…" indicator
// at all; while she thinks, the bar simply keeps the living
// line.)
function presenceState() {
  if (call) return { text: "In a call with you", dot: "#34d399", pulse: true, typing: false };
  if (CONFIG.backend !== "live") return { text: "Simulated (mock mode)", dot: "#f5b544", pulse: false, typing: false };
  if (llama.status === "running") return { text: presenceText, dot: "#34d399", pulse: true, typing: false };
  return { text: "Navi is not here — her model is disconnected.", dot: "#9fb3c8", pulse: false, typing: false };
}
// Her presence bar (Build 8): Navi's status leaves the Chat
// view and becomes frame chrome — one persistent element above
// every view, mounted once here and only ever REPAINTED, never
// re-created, so the node (and anything riding it) survives
// every view switch. Its right edge carries the pane button
// restored to its pre-#123 form (#115's button, Build 6): the
// panel glyph with aria-pressed, in the persistent top right on
// every page. presenceState() stays the single source for the
// dot and the line; the status column's own presence line
// inside the pane is untouched.
var presenceBarEl = document.getElementById("presence-bar");
var chatStatusToggleBtn = null;
function renderPresenceBar() {
  if (!presenceBarEl) return;
  var pres = presenceState();
  // The button's pressed state is READ OFF THE FRAME (Build 9,
  // #130): render() has just set #app's classes from the state
  // flags, and those classes are what actually show or hide the
  // column (desktop) and the dock (phone) — so the button
  // transcribes the rendered fact instead of re-deriving it
  // from a parallel flag expression, and its accent in CSS is
  // keyed to the same classes. One fact, one place: the button
  // cannot tell a different story from the screen.
  var framePressed = window.innerWidth <= 800 ? appEl.classList.contains("dock-open") : !appEl.classList.contains("no-status");
  // Build 15 (#161): in a case's own chat, the header says which
  // case this room belongs to — the running case, visible where
  // the work happens; the chip is the door back to the case.
  var caseChip = "";
  if (view === "Chat") {
    var ac = (typeof activeChat === "function") ? activeChat() : null;
    if (ac && ac.case) {
      var cname = String(ac.title || ac.case).replace(/^Case — /, "");
      caseChip = "<button class='ghost' id='presence-case-chip' aria-label='Open the case this chat belongs to' style='min-height:44px;padding:4px 10px;font-size:12px;white-space:nowrap'>⚖ Case: " + esc(cname) + "</button>";
    }
  }
  presenceBarEl.innerHTML = "<div class='chat-presence'><img class='presence-av' src='assets/navi-portrait.jpg' alt='Navi'><span class='dot" + (pres.pulse ? " pdot" : "") + "' style='background:" + pres.dot + "'></span><div style='min-width:0'><div class='chat-presence-name'>Navi</div><div class='chat-presence-line' id='chat-presence-line'>" + esc(pres.text) + "</div></div></div>"
    + caseChip
    + "<button class='ghost' id='chat-status-toggle' aria-pressed='" + (framePressed ? "true" : "false") + "' aria-label='Show or hide the status column' title='Status column'>" + statusPanelSvg() + "</button>";
  chatStatusToggleBtn = document.getElementById("chat-status-toggle");
  if (chatStatusToggleBtn) chatStatusToggleBtn.onclick = function () { toggleStatusColumn(); };
  var pcc = document.getElementById("presence-case-chip");
  if (pcc) pcc.onclick = function () { var a2 = activeChat(); if (a2 && a2.case) { caseDetailState = null; view = "CaseFiles"; refreshCaseDetail(a2.case); render(); } };
}
function statusPanelSvg() {
  return "<svg aria-hidden='true' focusable='false' width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><rect x='3' y='4' width='18' height='16' rx='2'/><path d='M15.5 4v16'/><rect x='15.5' y='4' width='5.5' height='16' fill='currentColor' stroke='none' opacity='0.45'/></svg>";
}
// One door logic for the status column (Build 6, #114/#115):
// on desktop it shows/hides the column; on a phone it drives
// the dock — the same state the top bar's dock button reads,
// so the two doors can never disagree.
function toggleStatusColumn() {
  if (window.innerWidth <= 800) { dockOpen = !dockOpen; if (dockOpen) statusPaneTab = "status"; }
  else statusHidden = !statusHidden;
  render();
}
function esc(s) { return String(s).replace(/&/g, "&amp;").replace(/</g, "&lt;"); }
function renderLlama() {
  var h = "<div class='stat'><span class='dot" + (llama.status === "running" ? " pdot" : "") + "' style='background:" + (llama.status === "running" ? "#34d399" : "#9fb3c8") + "'></span><span style='color:var(--dim)'>NAVI</span> <span class='mono' style='color:var(--dim);font-size:11px'>llamafile · port 8080</span><br>";
  if (llama.status === "running") {
    h += "<span class='mono' style='color:#34d399;font-weight:700'>Running</span><details style='margin-top:2px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Server details</summary><div style='margin-top:2px'><div style='color:var(--text);font-size:12px;font-weight:700'>Local Llamafile Server</div><div class='mono' style='color:var(--dim);font-size:12px'>http://localhost:8080</div><div style='color:var(--text);font-size:12px;font-weight:700;margin-top:4px'>Apex Web UI</div><div class='mono' style='color:var(--dim);font-size:12px'>http://localhost:" + BRIDGE_PORT + "</div><div style='color:var(--text);font-size:12px;font-weight:700;margin-top:4px'>Bridge</div><div class='mono' style='color:var(--dim);font-size:12px'>build " + BRIDGE_BUILD + (bridgeRoot ? " · " + esc(bridgeRoot) : "") + "</div></div></details><button class='ghost' id='llama-disc' style='margin-top:6px'>Disconnect</button>";
  } else {
    h += "<button class='ghost' id='llama-conn' style='border-color:#f87171;color:#f87171;font-weight:700;margin-top:4px'>Disconnected. Click to Connect.</button>";
    if (llama.error) h += "<div style='color:#f87171;font-size:12px;margin-top:4px'>" + esc(llama.error) + "</div>";
    h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>No model connected — chat is a plain command line until it connects.</div>";
  }
  return h + "</div>";
}
function wireLlama() {
  var conn = document.getElementById("llama-conn");
  if (conn) conn.onclick = function () {
    conn.textContent = "Starting navi.llamafile — first load can take a minute…"; conn.disabled = true;
    if (CONFIG.backend === "mock") {
      setTimeout(function () { llama.status = "running"; llama.error = null; render(); }, 250);
    } else {
      fetch("/api/llamafile/connect", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
        .then(function (r) { return r.json(); })
        .then(function (j) { llama.status = j.ok ? "running" : "stopped"; llama.error = j.ok ? null : (j.error || "No answer from navi.llamafile."); render(); })
        .catch(function () { llama.status = "stopped"; llama.error = "No answer from navi.llamafile — is bridge.py running?"; render(); });
    }
  };
  var disc = document.getElementById("llama-disc");
  if (disc) disc.onclick = function () { llama.status = "stopped"; llama.error = null; render(); };
}
function renderLocalNetwork() {
  var ps = publicState || CONFIG.publicServer || { enabled: false, activePublic: false, pending: false, lanUrl: "", accessCode: "" };
  // Build 2 (#204): the status-bar law — the Apex | Node
  // (local server) row shows only while the feature is on,
  // live, or mid-switch (the flip's own feedback); switched
  // off and settled, the row is ABSENT. Off = no trace.
  if (!ps.enabled && !ps.activePublic && !ps.pending) return "";
  // Build .65: the ON face — the face line, the address, the code —
  // stands only on a live report that the answering bridge is
  // serving the LAN right now. The saved choice alone no longer
  // opens the details (it kept the last-used address and code on
  // screen while the server was off), and in live mode a
  // last-known report from a bridge that has stopped answering
  // does not count either (publicReportLive above).
  var live = !!ps.activePublic && (CONFIG.backend !== "live" || publicReportLive);
  var pending = !!ps.pending;
  var hasCode = !!(ps.accessCode);
  var dot = pending ? "#f5b544" : live ? "#34d399" : "#9fb3c8";
  var stateText = pending ? "Switching\u2026" : live ? "On your network" : "This machine only";
  var h = "<div class='stat'><span class='dot' style='background:" + dot + "'></span><span style='color:var(--dim)'>APEX NODE</span> <span style='border:1px solid var(--line);border-radius:4px;padding:0 4px;font-size:10px;color:var(--dim)'>[local]</span><br>" +
    "<span class='v mono' style='font-weight:700'>" + esc(stateText) + "</span>" +
    "<details id='local-net-details'" + (localNetOpen ? " open" : "") + " style='margin-top:2px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Access details</summary><div style='margin-top:6px'>";
  if (live) {
    if (ps.lanUrl) {
      h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>ON YOUR PHONE, OPEN THIS ADDRESS</div>" +
        "<button class='ghost' id='local-net-copy-lan' aria-label='Copy the local network address " + esc(ps.lanUrl) + "' style='margin-top:4px;display:block;width:100%;text-align:left;font-weight:700;color:var(--cy);border-color:var(--cy);word-break:break-all;min-height:40px'>" + (localNetCopied === "lan" ? "Copied \u2014 " : "") + esc(ps.lanUrl) + "</button>";
    }
    if (hasCode) {
      h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em;margin-top:8px'>ACCESS CODE</div>" +
        "<div style='display:flex;gap:6px;align-items:center;flex-wrap:wrap;margin-top:4px'><span class='mono' style='border:1px solid var(--line);background:var(--panel2);padding:6px 10px;font-weight:700;letter-spacing:.2em'>" + esc(ps.accessCode) + "</span>" +
        "<button class='ghost' id='local-net-copy-code' aria-label='Copy the Apex | Node access code' style='min-height:40px'>" + (localNetCopied === "code" ? "Copied" : "Copy code") + "</button></div>";
    }
  } else {
    h += "<div style='color:var(--dim);font-size:12px'>Apex | Node is off. Its controls live on the DHD page, under the DHD options.</div>";
  }
  h += "<button class='ghost' id='local-net-settings' aria-label='Open the DHD page for the Apex | Node toggle and Regenerate' style='margin-top:8px;display:block;width:100%;text-align:center;font-weight:700;min-height:40px'>Open DHD page</button>" +
    (live ? "<div style='color:var(--dim);font-size:12px;margin-top:4px'>The Apex | Node toggle and Regenerate live there.</div>" : "") +
    "</div></details></div>";
  return h;
}
function commandBlockHtml(commands, label) {
  if (!commands || !commands.length) return "";
  var text = commands.join("\n");
  return "<div style='margin-top:8px'><div style='color:var(--dim);font-size:12px'>" + esc(label || "Manual commands — paste into Apex chat (CLI mode) or a terminal") + "</div>" +
    "<pre class='exec mono' style='margin:6px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>" + esc(text) + "</pre>" +
    "<button type='button' class='ghost cmd-copy' data-copy=\"" + encodeURIComponent(text) + "\" aria-label='Copy the manual commands' style='min-height:40px'>Copy commands</button></div>";
}
function wireCommandBlocks(root) {
  var scope = root || document;
  var buttons = scope.querySelectorAll ? scope.querySelectorAll(".cmd-copy") : [];
  buttons.forEach(function (b) {
    if (b.dataset.wired) return;
    b.dataset.wired = "1";
    b.onclick = function () {
      var text = decodeURIComponent(b.getAttribute("data-copy") || "");
      var done = function () { b.textContent = "Copied"; setTimeout(function () { b.textContent = "Copy commands"; }, 1800); };
      if (navigator.clipboard && navigator.clipboard.writeText) navigator.clipboard.writeText(text).then(done).catch(function () { done(); });
      else {
        var ta = document.createElement("textarea");
        ta.value = text; document.body.appendChild(ta); ta.select();
        try { document.execCommand("copy"); } catch (e) {}
        document.body.removeChild(ta); done();
      }
    };
  });
}
function operationCardHtml(op) {
  if (!op) return "";
  var icons = { done: ["✓", "var(--green)"], running: ["●", "var(--cy)"], waiting: ["◐", "#f5b544"], failed: ["✕", "#f87171"], pending: ["○", "var(--dim)"] };
  var h = "<section class='card op-card' data-op-id='" + esc(op.id) + "' aria-label='" + esc(op.title || "Operation") + "' style='margin-top:8px'><b>" + esc(op.title || "Operation") + "</b>";
  h += "<div style='color:var(--dim);font-size:12px;margin-top:2px'>" + (op.state === "done" ? "Complete" : op.state === "failed" ? "Failed" : op.state === "waiting" ? "Waiting for an action" : "Working") + (op.detail ? " — " + esc(op.detail) : "") + "</div>";
  (op.steps || []).forEach(function (s) {
    var ic = icons[s.state] || icons.pending;
    h += "<div style='display:flex;gap:8px;margin-top:8px;align-items:flex-start'><span aria-hidden='true' style='color:" + ic[1] + ";font-weight:700;line-height:1.4'>" + ic[0] + "</span><div style='min-width:0;flex:1'><b style='font-size:13px'>" + esc(s.name) + "</b>";
    if (s.detail) h += "<div style='color:var(--dim);font-size:12px'>" + esc(s.detail) + "</div>";
    if (s.output) h += "<details style='margin-top:4px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Step output</summary><pre class='exec mono' style='margin-top:6px;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-word'>" + esc(s.output) + "</pre></details>";
    if (s.commands && s.commands.length && (s.state === "failed" || s.state === "waiting")) h += commandBlockHtml(s.commands);
    if (s.id === "prerequisites" && s.state === "waiting") h += "<button type='button' class='act' data-op-install='" + esc(op.id) + "' aria-label='Install the missing DHD prerequisites' style='min-height:44px;margin-top:8px'>Install missing prerequisites</button>";
    h += "</div></div>";
  });
  if (op.kind === "badge-sync" && op.state === "waiting") h += "<div class='row' style='margin-top:10px'><button type='button' class='act' data-badge-apply='" + esc(op.id) + "' style='min-height:44px'>Confirm — land this sync</button> <button type='button' class='ghost' data-badge-dismiss='" + esc(op.id) + "' style='min-height:44px'>Not now</button></div>";
  return h + "</section>";
}
function wireOperationCards(root) {
  wireCommandBlocks(root);
  var bscope = root || document;
  if (bscope.querySelectorAll) {
    bscope.querySelectorAll("[data-badge-apply]").forEach(function (b) {
      if (b.dataset.wired) return;
      b.dataset.wired = "1";
      b.onclick = function () {
        var opId = b.getAttribute("data-badge-apply");
        b.disabled = true; b.textContent = "Landing it…";
        fetch("/api/github/badge-sync/apply", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ operationId: opId }) })
          .then(function (r) { return r.json(); })
          .then(function (j) {
            if (j && j.dashboard) adoptDashboardSlice(j.dashboard);
            fetch("/api/persona").then(function (r) { return r.json(); }).then(function (p) { if (p.soul) CONFIG.soul = p.soul; if (p.memory) CONFIG.memory = p.memory; }).catch(function () {});
            hydrateChatsFromServer();
            refreshOperation(opId);
          })
          .catch(function () { refreshOperation(opId); });
      };
    });
    bscope.querySelectorAll("[data-badge-dismiss]").forEach(function (b) {
      if (b.dataset.wired) return;
      b.dataset.wired = "1";
      b.onclick = function () {
        var opId = b.getAttribute("data-badge-dismiss");
        b.disabled = true;
        fetch("/api/github/badge-sync/dismiss", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ operationId: opId }) })
          .then(function () { refreshOperation(opId); })
          .catch(function () { refreshOperation(opId); });
      };
    });
  }
  var scope = root || document;
  if (!scope.querySelectorAll) return;
  scope.querySelectorAll("[data-op-install]").forEach(function (b) {
    if (b.dataset.wired) return;
    b.dataset.wired = "1";
    b.onclick = function () {
      b.disabled = true; b.textContent = "Installing…";
      fetch("/api/dhd/install", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ sudo_password: "" }) })
        .then(function (r) { return r.json(); })
        .then(function (j) {
          if (j && j.dhd) dhdState = j.dhd;
          if (j && !j.ok) {
            // The waiting operation is released as failed by the
            // bridge and its card repaints with commands; refresh
            // once more so the failure is what stays in view.
          }
          refreshOperation(b.getAttribute("data-op-install"));
        })
        .catch(function () { b.disabled = false; b.textContent = "Install missing prerequisites"; });
    };
  });
}
function refreshOperation(opId) {
  if (!opId || CONFIG.backend !== "live") return;
  fetch("/api/operations/" + encodeURIComponent(opId))
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (!j || !j.operation) return;
      var op = j.operation;
      if (dhdOperation && dhdOperation.id === opId) dhdOperation = op;
      messages.forEach(function (m) { if (m.operation && m.operation.id === opId) m.operation = op; });
      persist();
      document.querySelectorAll("[data-op-id='" + opId + "']").forEach(function (el) {
        var tmp = document.createElement("div");
        tmp.innerHTML = operationCardHtml(op);
        var fresh = tmp.firstElementChild;
        if (fresh) { el.replaceWith(fresh); wireOperationCards(fresh); }
      });
      if (view === "DHD" && dhdOperation && dhdOperation.id === opId && op.state !== "running" && op.state !== "waiting") {
        fetch("/api/status").then(function (r) { return r.json(); }).then(function (s) { if (s && s.dhd) { dhdState = s.dhd; dhdMsg = op.detail || ""; } render(); }).catch(function () { render(); });
        return;
      }
      if (op.state === "running" || op.state === "waiting") {
        if (opPollTimers[opId]) clearTimeout(opPollTimers[opId]);
        opPollTimers[opId] = setTimeout(function () { refreshOperation(opId); }, 800);
      }
    })
    .catch(function () {
      if (opPollTimers[opId]) clearTimeout(opPollTimers[opId]);
      opPollTimers[opId] = setTimeout(function () { refreshOperation(opId); }, 1500);
    });
}
function watchOperation(opId) {
  if (!opId) return;
  if (opPollTimers[opId]) clearTimeout(opPollTimers[opId]);
  opPollTimers[opId] = setTimeout(function () { refreshOperation(opId); }, 350);
}
function dhdPrereqCardHtml(d) {
  var p = d.prereqs;
  if (!p || !p.items) return "";
  var missing = p.items.filter(function (i) { return !i.ok; });
  var h = "<div class='card'><b>Prerequisites</b><p style='color:var(--dim)'>Installed with Apex on " + esc(p.label || "this machine") + " and checked again here. If a piece is ever removed, Apex can install it in this flow and then continue.</p>";
  p.items.forEach(function (i) {
    h += "<div style='display:flex;gap:8px;margin-top:7px;align-items:flex-start'><span class='dot' style='background:" + (i.ok ? "#34d399" : "#f5b544") + ";margin-top:5px'></span><div style='min-width:0'><b style='font-size:13px'>" + esc(i.name) + "</b><div style='color:var(--dim);font-size:12px'>" + esc(i.detail) + "</div>";
    if (!i.ok) h += commandBlockHtml(i.commands, i.installable ? "If the install does not work, run these exact commands" : "Manual remedy — exact commands for this machine");
    h += "</div></div>";
  });
  if (missing.some(function (i) { return i.installable; })) h += "<button class='act' id='dhd-install' aria-label='Install the missing DHD prerequisites' style='min-height:44px;margin-top:10px'>Install missing prerequisites</button>";
  var op = dhdOperation || d.lastOperation;
  if (op) { dhdOperation = op; h += operationCardHtml(op); }
  return h + "</div>";
}
function apexNodeCardHtml() {
  // Apex | Node (the home-network door; display name decided
  // 2026-11-08): its controls live on the DHD page, under the
  // DHD options - Settings keeps only a pointer. Same config
  // keys, same API paths, same fluid bridge-restart activation
  // as the control always had; only the home moved.
  var h = "";
      var ps = publicState || CONFIG.publicServer || { enabled: false, activePublic: false, bindHost: "127.0.0.1", pending: false, lanIp: "", lanUrl: "", accessCode: "", restart: { status: "idle", step: "", message: "" } };
      var rst = ps.restart || { status: "idle", step: "", message: "" };
      // Build .65: the row's live claim follows the same rule as
      // the status column's section — a last-known report from a
      // bridge that has stopped answering never reads as live.
      var rowLive = !!ps.activePublic && (CONFIG.backend !== "live" || publicReportLive);
      h += "<div class='card'><h3>Apex | Node <span style='border:1px solid var(--line);border-radius:4px;padding:0 5px;font-size:11px;color:var(--dim)'>[local]</span></h3>";
      h += "<p style='color:var(--dim)'>Turn this on and Apex serves its web page to your local network, so you can open her from a phone or tablet on the same network — same Apex, in the phone's browser. Off (the default), she answers only this machine. Flipping the switch is the whole job: the choice is saved in <span class='mono'>config/navi.config.json</span> and <b>Apex restarts her own bridge right then</b> so it takes effect — no relaunching the app yourself.</p>";
      h += "<p style='color:var(--am)'><b>Warning:</b> the access code is the knock on this door — a device that presents it still has to be approved (the first device approves itself), and an approved device can use Apex in full: chat, the command line, and settings included. Share the code like a house key.</p>";
      h += "<p><button type='button' id='pub-toggle' class='" + (ps.enabled ? "act" : "ghost") + "' aria-pressed='" + (ps.enabled ? "true" : "false") + "' aria-label='Apex | Node toggle'>" + (ps.enabled ? "Apex | Node: On" : "Apex | Node: Off") + "</button></p>";
      h += "<p style='color:var(--dim)'><b style='color:var(--text)'>" + (rowLive ? "On your network" : "This machine only — 127.0.0.1:" + BRIDGE_PORT) + "</b>";
      if (rst.status === "restarting" || publicRestartPolling) h += "<br><span style='color:var(--cy)'>Restarting the bridge…</span>";
      else if (rst.status === "failed") h += "<br><span style='color:#f87171'><b>Restart failed — step: " + esc(rst.step === "stop" ? "stopping the old bridge" : rst.step === "verify" ? "checking the new bridge's bind" : "the new bridge answering") + ".</b> " + esc(rst.message || "") + "</span>";
      else if (rst.status === "confirmed" && !ps.pending) h += "<br><span style='color:var(--green)'>" + esc(rst.message || "Live — the bridge that answers runs the saved choice.") + "</span>";
      else if (ps.pending) h += "<br><span style='color:var(--am)'>The saved choice is not live yet — the running bridge still uses the old setting. Press Restart bridge and Apex applies it herself.</span>";
      else h += "<br>The running bridge matches the saved choice.";
      h += "</p>";
      if ((rst.status === "failed" || (ps.pending && rst.status !== "restarting")) && !publicRestartPolling) h += "<p><button type='button' class='ghost' id='pub-restart' aria-label='Restart the bridge to apply the Apex | Node choice'>Restart bridge</button> <span style='color:var(--dim)'>Fallback if it still will not take: quit Apex fully and relaunch.</span></p>";
      if (ps.enabled || rowLive) {
        h += "<p style='color:var(--dim)'>Access code — shown only here, on this machine:<br><span class='mono' style='font-size:18px;letter-spacing:.2em;color:var(--text)'>" + esc(ps.accessCode || "—") + "</span> <button type='button' class='ghost' id='pub-regen' aria-label='Regenerate the Apex | Node access code'>Regenerate</button></p>";
        h += "<p style='color:var(--dim)'>A phone visiting Apex knocks with this code and a name for itself; the first phone approves itself, and every phone after that waits for an approval in the Door list. Regenerating makes a new code and stops new knocks with the old one — phones already approved keep their own sessions, and the Door list revokes them one by one.</p>";
        if (ps.lanUrl) h += "<p style='color:var(--dim)'>On your phone, open this address:<br><button type='button' class='act' id='pub-lan' data-url='" + esc(ps.lanUrl) + "' aria-label='Copy the LAN address " + esc(ps.lanUrl) + "'>" + esc(ps.lanUrl) + "</button></p>";
        h += "<p style='color:var(--dim)'>Same network, that address, then the code above. This machine never needs the code.</p>";
      }
      if (publicMsg) h += "<p style='color:var(--dim)'>" + esc(publicMsg) + "</p>";
      h += "</div>";
  return h;
}
function doorWhen(epochSeconds) {
  if (!epochSeconds) return "—";
  var d = new Date(epochSeconds * 1000);
  var mon = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"][d.getMonth()];
  return mon + " " + d.getDate() + ", " + ("0" + d.getHours()).slice(-2) + ":" + ("0" + d.getMinutes()).slice(-2);
}
function doorDevicesCardHtml() {
  // The Door — trusted devices (Build 2, #173). One renderer for
  // both homes: the DHD page (where the door's controls live) and
  // Settings → System → Door. Buttons carry the device id in a
  // data attribute, so the two homes never fight over element ids.
  var ps = publicState || CONFIG.publicServer || {};
  var devices = ps.devices || [];
  var pending = devices.filter(function (d) { return d.status === "pending"; });
  var trusted = devices.filter(function (d) { return d.status === "approved"; });
  var h = "<div class='card'><h3>The Door — trusted devices</h3>";
  h += "<p style='color:var(--dim)'>The access code no longer opens Apex by itself — it <b>knocks</b>. A new device that enters the code and names itself waits here as a pending device until a session that is already trusted approves it. An approved device gets its <b>own</b> session — 30 days, renewed while it stays in use — and can be revoked on its own, without changing the code under everyone else. The very first device at a door approves <b>itself</b> with the code, so a one-operator house is never locked out waiting for an approver who doesn't exist yet; from then on, every device knocks.</p>";
  if (doorMsg) h += "<p style='color:var(--dim)'>" + esc(doorMsg) + "</p>";
  if (pending.length) {
    h += "<p class='mono' style='color:var(--am);font-size:12px;margin-top:10px'>WAITING AT THE DOOR</p>";
    pending.forEach(function (d) {
      h += "<div style='border:1px solid var(--am);border-radius:8px;padding:10px 12px;margin-top:8px'><b>" + esc(d.name || "Unnamed device") + "</b> <span style='color:var(--dim);font-size:12px'>knocked " + esc(doorWhen(d.firstSeen)) + (d.userAgent ? " · " + esc(d.userAgent) : "") + "</span><div style='margin-top:8px'><button type='button' class='act' data-door-approve='" + esc(d.id) + "' style='min-height:44px'>Approve — let it in</button> <button type='button' class='ghost' data-door-deny='" + esc(d.id) + "' style='min-height:44px'>Deny</button></div></div>";
    });
  }
  h += "<p class='mono' style='color:var(--dim);font-size:12px;margin-top:12px'>TRUSTED DEVICES</p>";
  if (!trusted.length) h += "<p style='color:var(--dim)'>None yet. The first device to knock with the code lets itself in and appears here.</p>";
  trusted.forEach(function (d) {
    h += "<div style='border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-top:8px'><b>" + esc(d.name || "Unnamed device") + "</b>" + (d.expired ? " <span style='color:#f87171;font-size:12px'>EXPIRED</span>" : "") + "<div style='color:var(--dim);font-size:12px'>First seen " + esc(doorWhen(d.firstSeen)) + " · last seen " + esc(doorWhen(d.lastSeen)) + " · " + (d.expired ? "expired " : "expires ") + esc(doorWhen(d.expiresAt)) + "</div><div style='margin-top:8px'><button type='button' class='ghost' data-door-revoke='" + esc(d.id) + "' style='min-height:44px'>Revoke this device</button></div></div>";
  });
  h += "</div>";
  return h;
}
function wireDoorDevices() {
  var act = function (kind, id) {
    if (CONFIG.backend !== "live") {
      var ps = CONFIG.publicServer = CONFIG.publicServer || { devices: [] };
      var list = ps.devices || [];
      if (kind === "approve") list.forEach(function (d) { if (d.id === id) { d.status = "approved"; d.expiresAt = Math.floor(Date.now() / 1000) + 30 * 24 * 3600; } });
      if (kind === "deny" || kind === "revoke") ps.devices = list.filter(function (d) { return d.id !== id; });
      publicState = ps;
      doorMsg = "Simulated: the device was " + (kind === "approve" ? "approved" : kind === "deny" ? "denied" : "revoked") + " (mock mode).";
      render();
      return;
    }
    fetch("/api/settings/public/devices/" + kind, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ id: id }) })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.publicServer) publicState = j.publicServer; doorMsg = j && j.ok ? "Done — the list above is the door as it stands now." : ((j && j.error) || "The bridge refused that."); render(); })
      .catch(function () { doorMsg = "The bridge did not answer — is it running?"; render(); });
  };
  document.querySelectorAll("[data-door-approve]").forEach(function (b) { b.onclick = function () { act("approve", b.getAttribute("data-door-approve")); }; });
  document.querySelectorAll("[data-door-deny]").forEach(function (b) { b.onclick = function () { act("deny", b.getAttribute("data-door-deny")); }; });
  document.querySelectorAll("[data-door-revoke]").forEach(function (b) { b.onclick = function () { act("revoke", b.getAttribute("data-door-revoke")); }; });
}
function dhdHtml() {
  var d = dhdViewState();
  var ps = publicState || CONFIG.publicServer || { accessCode: "" };
  var code = d.accessCode || ps.accessCode || "";
  var modeLine = !d.active ? "Off" : d.mode === "domain" ? "Domain certificate (Let's Encrypt)" : "Self-signed certificate";
  var h = "<h3>Apex | DHD - the way home</h3>";
  h += "<p style='color:var(--dim);max-width:640px'>DHD takes the same Apex you use at home and gives it a public door: Apex itself stands up a reverse proxy with TLS on this machine - no Tailscale, no tunnel service, no Certbot - and you reach her from anywhere with one address and the access code. One click is the whole setup.</p>";
  h += "<div class='card'><div class='row' style='justify-content:space-between'><b>DHD - public access <span style='border:1px solid var(--line);border-radius:4px;padding:0 5px;font-size:11px;color:var(--dim)'>[public]</span></b>" +
    "<button class='ghost' id='dhd-toggle' role='switch' aria-checked='" + (d.enabled ? "true" : "false") + "' aria-label='DHD - public access to Apex from anywhere' style='min-width:64px;min-height:44px;font-weight:700;border-color:" + (d.enabled ? "var(--green)" : "var(--line)") + ";color:" + (d.enabled ? "var(--green)" : "var(--dim)") + "'>" + (d.enabled ? "ON" : "OFF") + "</button></div>" +
    "<p class='mono' style='font-weight:700;color:" + (d.active ? "var(--green)" : "var(--text)") + "'>" + (d.active ? "On - " + modeLine : d.enabled ? "Saved on - bringing the door up..." : "Off") + "</p>";
  if (dhdMsg) h += "<p style='color:var(--dim)'>" + esc(dhdMsg) + "</p>";
  if (d.lastError) h += "<p style='color:#f5b544'>" + esc(d.lastError) + "</p>";
  h += "<p><button class='ghost' id='dhd-docs' aria-label='Open the DHD documentation page' style='min-height:40px'>Read the DHD guide in Docs</button></p></div>";
  h += dhdPrereqCardHtml(d);
  h += "<div class='card'><b>Status</b><div class='mono' style='margin-top:8px;font-size:12px;line-height:1.9'>" +
    "MODE<br><span style='color:var(--text);font-size:14px'>" + esc(modeLine) + "</span><br>" +
    "PUBLIC ADDRESS<br>" + (d.publicVerified && d.publicUrl ? "<span style='color:var(--cy);font-size:14px;word-break:break-all'>" + esc(d.publicUrl) + "</span> <span style='color:var(--green);font-size:12px'>- outside reach verified</span>" : d.publicUrl ? "<span style='color:var(--cy);font-size:14px;word-break:break-all'>" + esc(d.publicUrl) + "</span> <span style='color:var(--am);font-size:12px'>- outside reach unverified</span>" : d.active ? "<span style='color:var(--text);font-size:14px'>No public address yet - the outside door is not set up</span>" : "<span style='color:var(--text);font-size:14px'>- DHD is off</span>") + "<br>" +
    "ON THIS NETWORK (LAN)<br>" + (d.lanUrl ? "<span style='color:var(--cy);font-size:14px;word-break:break-all'>" + esc(d.lanUrl) + "</span> <span style='color:var(--dim);font-size:12px'>- works only on this network</span>" : "<span style='color:var(--text);font-size:14px'>-</span>") + "<br>" +
    "OUTSIDE REACH<br><span style='color:var(--text);font-size:14px'>" + (d.publicVerified ? "Verified - the domain resolves to this machine and its certificate is live." : d.active ? "Unverified - " + esc(d.nextStep || "Forward ports 443 and 80 on your router to this machine, or add your domain.") + (d.wanNote ? " " + esc(d.wanNote) : "") : "-") + "</span><br>" +
    "CERTIFICATE<br><span style='color:var(--text);font-size:14px'>" + (d.active ? (d.certPending ? "Domain saved - certificate pending (door answering on the self-signed certificate meanwhile)" : d.certExpiresAt ? "Expires " + esc(String(d.certExpiresAt).slice(0, 10)) + (d.certDaysLeft != null ? " - " + d.certDaysLeft + " days left" : "") + (d.mode === "domain" ? " - Apex renews it on its own at 30 days or less" : "") : "Self-signed, generated by Apex") : "-") + "</span><br>" +
    "THE DOOR ITSELF<br><span style='color:var(--text);font-size:14px'>Proxy on 80/443, forwarding to the bridge on 127.0.0.1:" + BRIDGE_PORT + ". In DHD mode the bridge never binds publicly - the proxy is the only public face, and it serves nothing before the code.</span></div></div>";
  h += "<div class='card'><b>Access code - the same key as home</b><p style='color:var(--dim)'>One code guards both doors: Apex | Node and DHD use the same access code. It is shown only here, on the machine Apex runs on. Since Build 2 the code <b>knocks</b> rather than opens: a device that presents it waits for an approval in the Door list below (the very first device approves itself), and an approved device holds its own 30-day session. Share the code like a house key all the same - it is still the only way to knock. A session that comes in through DHD is a full session, the Updater included, end to end.</p>" +
    "<div class='row'><span class='mono' style='border:1px solid var(--line);background:var(--panel2);padding:8px 12px;font-weight:700;letter-spacing:.2em;font-size:16px'>" + (code ? (dhdCodeShown ? esc(code) : "••••-••••") : "No code yet - it is generated when a door first opens") + "</span>" +
    "<button class='ghost' id='dhd-code-show' aria-label='" + (dhdCodeShown ? "Hide the access code" : "Show the access code") + "' style='min-height:40px'>" + (dhdCodeShown ? "Hide" : "Show") + "</button>" +
    "<button class='ghost' id='dhd-code-regen' aria-label='Regenerate the access code - new knocks need the new code; approved devices keep their own sessions' style='min-height:40px'>Regenerate</button></div>" +
    "<p style='color:var(--dim);font-size:12px'>Regenerating stops new knocks with the old code at once. Devices already approved keep their own tokens — revoke those one by one in the Door list below.</p></div>";
  h += doorDevicesCardHtml();
  h += "<div class='card'><b>Your domain - optional, any time</b><p style='color:var(--dim)'>DHD never needs a domain: without one it runs on the self-signed certificate and your browser shows a warning that is expected (the Docs page explains it). Add your own domain whenever you like and Apex obtains its real Let's Encrypt certificate with its own built-in ACME client - no Certbot - proving the domain over port 80, then renewing on its own. Your router still has to forward 80/443 to this machine; the guide walks through it.</p>" +
    "<div class='row'><input class='grow' id='dhd-domain' type='text' placeholder='e.g. apex.example.com - blank is fine' aria-label='Custom domain for DHD' value='" + esc(d.domain || "") + "' style='min-height:44px'><button class='act' id='dhd-domain-save' aria-label='Save the DHD domain' style='min-height:44px'>Save domain</button></div>" +
    (d.domain && d.certPending ? "<p style='margin-top:8px'><button class='act' id='dhd-cert' aria-label='Run the certificate step for the saved domain' style='min-height:40px'>Get " + esc(d.domain) + "'s certificate</button></p><p style='color:var(--dim);font-size:12px'>The certificate step: account key, order, HTTP-01 challenge served through the Apex proxy on port 80, finalize, install, proxy reloads with the real certificate.</p>" : "") +
    "</div>";
  if (CONFIG.backend !== "live") h += "<div class='card'>In mock mode this page is simulated - the live bridge runs the proxy, certificates, and ACME client for real.</div>";
  h += apexNodeCardHtml();
  return h;
}
function wireDhd() {
  wireOperationCards(document);
  wireDoorDevices();
  wireApexNode();
  if (dhdOperation && (dhdOperation.state === "running" || dhdOperation.state === "waiting")) watchOperation(dhdOperation.id);
  var install = document.getElementById("dhd-install");
  if (install) install.onclick = function () {
    if (CONFIG.backend !== "live") { dhdMsg = "Simulated: prerequisites installed; the enable can continue."; render(); return; }
    install.disabled = true; install.textContent = "Installing…";
    fetch("/api/dhd/install", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ sudo_password: "" }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (j && j.dhd) dhdState = j.dhd;
        if (j && j.operation) { dhdOperation = j.operation; watchOperation(j.operation.id); }
        dhdMsg = j && j.ok ? "Prerequisites installed and verified." : "A prerequisite install failed — its exact manual commands are in the Prerequisites card.";
        render();
      })
      .catch(function () { dhdMsg = "The bridge did not answer."; render(); });
  };
  var t = document.getElementById("dhd-toggle");
  if (t) t.onclick = function () {
    var d = dhdViewState();
    var next = !d.enabled;
    if (CONFIG.backend !== "live") {
      CONFIG.dhd = CONFIG.dhd || { enabled: false, active: false, domain: "", mode: "off", status: "off", publicUrl: "", lanIp: "", lanUrl: "", wanIp: "", wanState: "undetected", wanNote: "", publicVerified: false, outsideReach: "none", nextStep: "", domainResolvesHere: false, certExpiresAt: null, certDaysLeft: null, certPending: false, lastError: "" };
      CONFIG.dhd.enabled = next;
      CONFIG.dhd.active = next;
      CONFIG.dhd.mode = next ? (CONFIG.dhd.domain ? "domain" : "self-signed") : "off";
      CONFIG.dhd.status = next ? (CONFIG.dhd.domain && CONFIG.dhd.certPending ? "degraded" : "healthy") : "off";
      CONFIG.dhd.domainResolvesHere = next && !!CONFIG.dhd.domain && !CONFIG.dhd.certPending;
      CONFIG.dhd.publicVerified = next && CONFIG.dhd.mode === "domain" && CONFIG.dhd.domainResolvesHere;
      CONFIG.dhd.lanIp = next ? "192.168.1.50" : "";
      CONFIG.dhd.lanUrl = next ? "https://192.168.1.50" : "";
      CONFIG.dhd.wanIp = next ? "192.0.2.44" : "";
      CONFIG.dhd.wanState = next ? "public" : "undetected";
      CONFIG.dhd.wanNote = "";
      CONFIG.dhd.outsideReach = next ? (CONFIG.dhd.publicVerified ? "verified" : "unverified") : "none";
      CONFIG.dhd.nextStep = next && !CONFIG.dhd.publicVerified ? "Forward ports 443 and 80 on your router to this machine, or add your domain - outside reach stays unverified until the public path is proven." : "";
      CONFIG.dhd.publicUrl = next ? "https://" + (CONFIG.dhd.publicVerified ? CONFIG.dhd.domain : "192.0.2.44") : "";
      dhdState = CONFIG.dhd;
      dhdMsg = next ? "Simulated: the proxy answers on 443 and the local check passed (answered on this machine). Outside reach is shown separately in Status." : "DHD is off.";
      render();
      return;
    }
    fetch("/api/dhd/toggle", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ enabled: next, sudo_password: "" }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (j && j.dhd) dhdState = j.dhd;
        if (next && j && j.operation) {
          dhdOperation = j.operation;
          dhdMsg = "Working — the operation card below is the live record.";
          render();
          watchOperation(j.operation.id);
        } else {
          dhdOperation = null;
          dhdMsg = "DHD is off.";
          render();
        }
      })
      .catch(function () { dhdMsg = "The bridge did not answer - is it running?"; render(); });
  };
  var docs = document.getElementById("dhd-docs");
  if (docs) docs.onclick = function () { docsPage = "dhd"; view = "Docs"; render(); };
  var show = document.getElementById("dhd-code-show");
  if (show) show.onclick = function () { dhdCodeShown = !dhdCodeShown; render(); };
  var regen = document.getElementById("dhd-code-regen");
  if (regen) regen.onclick = function () {
    if (CONFIG.backend !== "live") { dhdMsg = "Simulated: the code is regenerated from Settings in mock mode."; render(); return; }
    fetch("/api/settings/public/regenerate", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.publicServer) publicState = j.publicServer; dhdMsg = "New code saved - new knocks need the new code. Devices already approved keep their own sessions; revoke them in the Door list if you want them out."; render(); })
      .catch(function () { dhdMsg = "The bridge did not answer."; render(); });
  };
  var save = document.getElementById("dhd-domain-save");
  if (save) save.onclick = function () {
    var inp = document.getElementById("dhd-domain");
    var domain = inp ? inp.value.trim() : "";
    if (CONFIG.backend !== "live") {
      CONFIG.dhd = CONFIG.dhd || { enabled: false, active: false, domain: "", mode: "off", status: "off", publicUrl: "", lanIp: "", lanUrl: "", wanIp: "", wanState: "undetected", wanNote: "", publicVerified: false, outsideReach: "none", nextStep: "", domainResolvesHere: false, certExpiresAt: null, certDaysLeft: null, certPending: false, lastError: "" };
      CONFIG.dhd.domain = domain;
      if (CONFIG.dhd.active && domain) CONFIG.dhd.certPending = true;
      dhdState = CONFIG.dhd;
      dhdMsg = domain ? "Domain saved: " + domain + " (simulated)." : "Domain cleared (simulated).";
      render();
      return;
    }
    fetch("/api/dhd/domain", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ domain: domain }) })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.dhd) dhdState = j.dhd; dhdMsg = domain ? "Domain saved: " + domain + ". Its certificate step runs in the background while the door keeps answering." : "Domain cleared."; render(); })
      .catch(function () { dhdMsg = "The bridge did not answer."; render(); });
  };
  var cert = document.getElementById("dhd-cert");
  if (cert) cert.onclick = function () {
    if (CONFIG.backend !== "live") {
      CONFIG.dhd.certPending = false; CONFIG.dhd.mode = "domain"; CONFIG.dhd.status = "healthy";
      CONFIG.dhd.certExpiresAt = new Date(Date.now() + 90 * 86400000).toISOString(); CONFIG.dhd.certDaysLeft = 90;
      dhdState = CONFIG.dhd; dhdMsg = "Simulated: certificate installed and the proxy reloaded with it."; render(); return;
    }
    dhdMsg = "Running the certificate step - order, challenge on port 80, finalize, install...";
    render();
    fetch("/api/dhd/cert", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.dhd) dhdState = j.dhd; dhdMsg = j && j.ok ? "Certificate installed - the proxy reloaded with the real certificate." : (dhdState && dhdState.lastError) || "The certificate step did not finish - see the status line."; render(); })
      .catch(function () { dhdMsg = "The bridge did not answer."; render(); });
  };
}
function dhdViewState() {
  return dhdState || CONFIG.dhd || { enabled: false, active: false, domain: "", mode: "off", status: "off", publicUrl: "", lanIp: "", lanUrl: "", wanIp: "", wanState: "undetected", wanNote: "", publicVerified: false, outsideReach: "none", nextStep: "", domainResolvesHere: false, certExpiresAt: null, certDaysLeft: null, certPending: false, lastError: "" };
}
function renderDhdStatus() {
  var d = dhdViewState();
  // Build 2 (#204): the status-bar law — DHD off means the
  // DHD row is ABSENT from the status column, not an "Off"
  // placeholder. Off = no trace.
  if (!d.active) return "";
  var dot = d.status === "healthy" ? "#34d399" : (d.status === "degraded" || d.status === "starting") ? "#f5b544" : "#9fb3c8";
  var line = !d.active ? "Off - no public door" : d.status === "degraded" ? "On - domain saved, certificate pending" : d.publicVerified ? "On - " + d.domain + ", reachable anywhere" : "On - listening, outside reach unverified";
  var h = "<div class='stat'><span class='dot" + (d.active ? " pdot" : "") + "' style='background:" + dot + "'></span><span style='color:var(--dim)'>APEX DHD</span><br>" +
    "<span class='v mono' style='font-weight:700'>" + esc(line) + "</span>";
  h += "<details id='dhd-details'" + (dhdDetailsOpen ? " open" : "") + " style='margin-top:2px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Access details</summary><div style='margin-top:6px'>";
  if (d.active && d.publicVerified && d.publicUrl) {
    h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>FROM ANYWHERE, OPEN THIS ADDRESS</div>" +
      "<button class='ghost' id='dhd-copy-url' aria-label='Copy the public DHD address " + esc(d.publicUrl) + "' style='margin-top:4px;display:block;width:100%;text-align:left;font-weight:700;color:var(--cy);border-color:var(--cy);word-break:break-all;min-height:44px'>" + (dhdUrlCopied ? "Copied \u2014 " : "") + esc(d.publicUrl) + "</button>";
  } else if (d.active && d.publicUrl) {
    h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>PUBLIC ADDRESS - OUTSIDE REACH NOT YET PROVEN</div>" +
      "<button class='ghost' id='dhd-copy-url' aria-label='Copy the public DHD address " + esc(d.publicUrl) + "' style='margin-top:4px;display:block;width:100%;text-align:left;font-weight:700;color:var(--cy);border-color:var(--cy);word-break:break-all;min-height:44px'>" + (dhdUrlCopied ? "Copied \u2014 " : "") + esc(d.publicUrl) + "</button>" +
      "<div style='color:var(--dim);font-size:12px;margin-top:4px'>" + esc(d.nextStep || "") + "</div>";
  } else if (d.active) {
    h += "<div style='color:var(--dim);font-size:12px'>No public address yet - outside reach unverified. " + esc(d.nextStep || "") + "</div>";
  } else {
    h += "<div style='color:var(--dim);font-size:12px'>Not set up yet - DHD is off, so there is no public address. Turn it on from the DHD page and the address will live here.</div>";
  }
  if (d.active && d.lanUrl) {
    h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em;margin-top:8px'>ON THIS NETWORK (LAN)</div>" +
      "<button class='ghost' id='dhd-copy-lan' aria-label='Copy the LAN DHD address " + esc(d.lanUrl) + "' style='margin-top:4px;display:block;width:100%;text-align:left;font-weight:700;color:var(--cy);border-color:var(--cy);word-break:break-all;min-height:44px'>" + esc(d.lanUrl) + "</button>" +
      "<div style='color:var(--dim);font-size:12px;margin-top:2px'>Works only on this network.</div>";
  }
  var modeLine = !d.active ? "Off" : d.mode === "domain" ? "Domain certificate (Let's Encrypt)" : "Self-signed certificate";
  var certLine = !d.active ? "-" : d.certPending ? "Domain saved - certificate pending; the self-signed door answers meanwhile" : d.certExpiresAt ? "Expires " + String(d.certExpiresAt).slice(0, 10) + (d.certDaysLeft != null ? " - " + d.certDaysLeft + " days left" : "") : "Self-signed, generated by Apex";
  h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em;margin-top:8px'>MODE</div><div class='mono' style='font-size:12px'>" + esc(modeLine) + "</div>" +
    "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em;margin-top:8px'>CERTIFICATE</div><div style='font-size:12px'>" + esc(certLine) + "</div>" +
    "</div></details>";
  h += "<button class='ghost' id='dhd-open' aria-label='Open the DHD page - public access to Apex' style='margin-top:6px;display:block;width:100%;text-align:center;font-weight:700;min-height:44px'>Open DHD</button></div>";
  return h;
}
function wireApexNode() {
    // Apex | Node card (build .59; fluid activation .61): the
    // toggle saves the choice through the bridge AND starts the
    // bridge restart that applies it; the row then polls for the
    // relaunched bridge's confirmation (pollPublicRestart).
    // Regenerate persists the same way it always did.
    var pubToggle = document.getElementById("pub-toggle");
    if (pubToggle) pubToggle.onclick = function () {
      var ps = publicState || CONFIG.publicServer || { enabled: false };
      var next = !ps.enabled;
      if (CONFIG.backend !== "live") {
        CONFIG.publicServer = CONFIG.publicServer || { enabled: false, activePublic: false, bindHost: "127.0.0.1", pending: false, lanIp: "", lanUrl: "", accessCode: "", restart: { status: "idle", step: "", message: "" } };
        CONFIG.publicServer.enabled = next;
        CONFIG.publicServer.pending = next !== CONFIG.publicServer.activePublic;
        if (next && !CONFIG.publicServer.accessCode) CONFIG.publicServer.accessCode = "SIMULATED";
        if (CONFIG.publicServer.pending) {
          CONFIG.publicServer.restart = { status: "restarting", step: "restart", message: "Restarting the bridge…" };
          publicMsg = "Restarting the bridge… (mock mode — simulated).";
          render();
          var target = next;
          setTimeout(function () {
            CONFIG.publicServer.activePublic = target;
            CONFIG.publicServer.bindHost = target ? "0.0.0.0" : "127.0.0.1";
            CONFIG.publicServer.pending = false;
            CONFIG.publicServer.restart = { status: "confirmed", step: "", message: target ? "Restart complete — the bridge that answers now reports Apex | Node — on your network." : "Restart complete — the bridge that answers now reports This machine only." };
            saveCfg();
            publicMsg = CONFIG.publicServer.restart.message;
            render();
          }, 1200);
        } else {
          CONFIG.publicServer.restart = { status: "confirmed", step: "", message: "The bridge that answers already runs this choice — no restart was needed." };
          saveCfg();
          publicMsg = CONFIG.publicServer.restart.message;
          render();
        }
        return;
      }
      fetch("/api/settings/public", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ enabled: next }) }).then(function (r) { return r.json(); }).then(function (j) { publicReportLive = !!(j && j.publicServer); if (j && j.publicServer) publicState = j.publicServer; publicMsg = publicState && publicState.pending ? "Restarting the bridge…" : "Saved — the bridge that answers already runs this choice."; render(); if (publicState && publicState.pending) pollPublicRestart(); }).catch(function () { publicReportLive = false; publicMsg = "The bridge did not answer — is it running?"; render(); });
    };
    var pubRestart = document.getElementById("pub-restart");
    if (pubRestart) pubRestart.onclick = function () {
      if (CONFIG.backend !== "live") {
        var cps = CONFIG.publicServer || { enabled: false, activePublic: false };
        cps.restart = { status: "restarting", step: "restart", message: "Restarting the bridge…" };
        publicMsg = "Restarting the bridge… (mock mode — simulated).";
        render();
        setTimeout(function () {
          cps.activePublic = cps.enabled;
          cps.bindHost = cps.enabled ? "0.0.0.0" : "127.0.0.1";
          cps.pending = false;
          cps.restart = { status: "confirmed", step: "", message: cps.enabled ? "Restart complete — the bridge that answers now reports Apex | Node — on your network." : "Restart complete — the bridge that answers now reports This machine only." };
          saveCfg();
          publicMsg = cps.restart.message;
          render();
        }, 1200);
        return;
      }
      fetch("/api/settings/public/restart", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).then(function (r) { return r.json(); }).then(function (j) { publicReportLive = !!(j && j.publicServer); if (j && j.publicServer) publicState = j.publicServer; publicMsg = "Restarting the bridge…"; render(); pollPublicRestart(); }).catch(function () { publicReportLive = false; publicMsg = "The bridge did not answer — is it running?"; render(); });
    };
    var pubRegen = document.getElementById("pub-regen");
    if (pubRegen) pubRegen.onclick = function () {
      if (CONFIG.backend !== "live") {
        CONFIG.publicServer = CONFIG.publicServer || { enabled: true, activePublic: false, bindHost: "127.0.0.1", pending: false, lanIp: "", lanUrl: "", accessCode: "" };
        CONFIG.publicServer.accessCode = "MOCK-" + Math.floor(1000 + Math.random() * 9000);
        saveCfg();
        publicMsg = "New code (mock mode — simulated). New knocks need the new code; approved devices keep their own sessions.";
        render();
        return;
      }
      fetch("/api/settings/public/regenerate", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).then(function (r) { return r.json(); }).then(function (j) { publicReportLive = !!(j && j.publicServer); if (j && j.publicServer) publicState = j.publicServer; publicMsg = "New code saved — new knocks need the new code. Devices already approved keep their own sessions; revoke them in the Door list if you want them out."; render(); }).catch(function () { publicReportLive = false; publicMsg = "The bridge did not answer — is it running?"; render(); });
    };
    var pubLan = document.getElementById("pub-lan");
    if (pubLan) pubLan.onclick = function () {
      var url = pubLan.getAttribute("data-url") || "";
      var done = function () { publicMsg = "Copied " + url + " — open it on your phone, on the same network, and enter the access code."; render(); };
      if (navigator.clipboard && navigator.clipboard.writeText) navigator.clipboard.writeText(url).then(done).catch(function () { publicMsg = url; render(); });
      else { publicMsg = url; render(); }
    };
}
function wireDhdStatus() {
  var b = document.getElementById("dhd-open");
  if (b) b.onclick = function () { view = "DHD"; dockOpen = false; render(); };
  var det = document.getElementById("dhd-details");
  if (det) det.addEventListener("toggle", function () { dhdDetailsOpen = det.open; });
  var cp = document.getElementById("dhd-copy-url");
  if (cp) cp.onclick = function () {
    var d = dhdViewState();
    if (!d.publicUrl) return;
    var done = function () { dhdUrlCopied = true; render(); setTimeout(function () { if (dhdUrlCopied) { dhdUrlCopied = false; render(); } }, 2000); };
    if (navigator.clipboard && navigator.clipboard.writeText) navigator.clipboard.writeText(d.publicUrl).then(done).catch(function () {});
  };
  var cpl = document.getElementById("dhd-copy-lan");
  if (cpl) cpl.onclick = function () {
    var d = dhdViewState();
    if (!d.lanUrl) return;
    if (navigator.clipboard && navigator.clipboard.writeText) navigator.clipboard.writeText(d.lanUrl).catch(function () {});
  };
}
function meshViewState() {
  // One shape for both sources: the bridge's /api/status mesh
  // report (live) and the baked CONFIG.mesh block (mock mode and
  // the first paint before the first poll lands).
  if (meshState) return meshState;
  var m = CONFIG.mesh || {};
  var b = m.borrow || {}, l = m.lend || {};
  return {
    enabled: !!m.enabled, role: m.role || "both", nodeName: m.nodeName || "", nodeId: m.nodeId || "",
    publicEndpoint: m.publicEndpoint || "",
    listenPorts: (m.listenPorts && m.listenPorts.length) ? m.listenPorts : [8765, 8443, 2475],
    tunnelPublicKey: m.tunnelPublicKey || "",
    borrowEnabled: b.enabled !== false, graceMs: b.graceMs != null ? b.graceMs : 2000,
    lendEnabled: !!l.enabled,
    lend: { maxSharePercent: l.maxSharePercent != null ? l.maxSharePercent : 25, maxGenerationsPerHour: l.maxGenerationsPerHour != null ? l.maxGenerationsPerHour : 20, perPeerPerHour: l.perPeerPerHour != null ? l.perPeerPerHour : 10, idleOnly: l.idleOnly !== false, idleAfterSeconds: l.idleAfterSeconds != null ? l.idleAfterSeconds : 60 },
    ledgerInBackup: !!m.ledgerInBackup, certNoticeSeen: !!m.certNoticeSeen, borrowing: false, lendingNow: false,
    headline: "",
    viaDhd: false,
    state: m.enabled ? ((b.peers || []).length ? "on" : "no-peers") : "off", warning: "",