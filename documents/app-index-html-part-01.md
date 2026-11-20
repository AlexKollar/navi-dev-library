# app/index.html — part 1 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 1–2421 of app/index.html.

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>Apex</title>
<link rel="icon" type="image/png" sizes="32x32" href="favicon-32.png">
<link rel="apple-touch-icon" sizes="180x180" href="apple-touch-icon.png">
<style>
  :root { color-scheme:dark; --bg:#0a0e12; --panel:#10161d; --panel2:#0d1218; --line:#22303d; --text:#e6f1f5; --dim:#9fb3c8; --cy:#22d3ee; --blue:#58a6ff; --am:#f5b544; --red:#f87171; --green:#34d399; --ver-alpha:#f87171; --ver-beta:#f5b544; --ver-stable:#22d3ee; --syn-cmd:#4ade80; --syn-bad:#ff8a8a; --syn-flag:#22d3ee; --syn-str:#fbbf24; --syn-path:#93c5fd; --syn-kw:#f9a8d4; --syn-num:#c4b5fd; --syn-op:#e879f9; --exec-bg:#000000; }
  /* Curated themes (Settings): the same variables, twelve palettes.
     Nocturne Row is the default (issue #118); the others swap the
     whole variable set on <body data-theme>, so every view — selects and
     inputs included — follows the one choice. Nord is the dark Nord
     family: Polar Night backgrounds, Snow Storm text, Frost accents,
     muted Aurora highlights. Code and terminal surfaces stay dark
     in the light theme (their text is pinned light below). */
  body[data-theme="light"] { color-scheme:light; --bg:#e8edf1; --panel:#f7fafc; --panel2:#eef3f6; --line:#b3c3cf; --text:#12212b; --dim:#43586a; --cy:#0284c7; --blue:#1d6fd1; --am:#b45309; --red:#dc2626; --green:#059669; --ver-alpha:#dc2626; --ver-beta:#b45309; --ver-stable:#0284c7; --syn-cmd:#4ade80; --syn-bad:#ff8a8a; --syn-flag:#22d3ee; --syn-str:#fbbf24; --syn-path:#93c5fd; --syn-kw:#f9a8d4; --syn-num:#c4b5fd; --syn-op:#e879f9; --exec-bg:#000000; }
  body[data-theme="light"] .hlayer { --syn-cmd:#047857; --syn-bad:#b91c1c; --syn-flag:#0369a1; --syn-str:#92400e; --syn-path:#1d4ed8; --syn-kw:#a21caf; --syn-num:#6d28d9; --syn-op:#9d174d; }
  body[data-theme="light"] .ed-code pre, body[data-theme="light"] .hlayer { color:#e6f1f5; }
  body[data-theme="light"] .ed-gutter, body[data-theme="light"] .ed-mini pre { color:#9fb3c8; }
  body[data-theme="contrast"] { --bg:#000000; --panel:#0c0d0f; --panel2:#14161a; --line:#7f95ab; --text:#ffffff; --dim:#d7e3ee; --cy:#2fd8ff; --am:#ffc933; --red:#ff7a7a; --green:#4ade80; --ver-alpha:#ff7a7a; --ver-beta:#ffc933; --ver-stable:#2fd8ff; --syn-cmd:#4ade80; --syn-bad:#ff9a9a; --syn-flag:#2fd8ff; --syn-str:#ffc933; --syn-path:#a5c8ff; --syn-kw:#f0abfc; --syn-num:#ddd6fe; --syn-op:#fda4af; --exec-bg:#000000; }
  body[data-theme="nord"] { --bg:#2e3440; --panel:#3b4252; --panel2:#434c5e; --line:#4c566a; --text:#eceff4; --dim:#a9b4c6; --cy:#88c0d0; --am:#ebcb8b; --red:#d98a94; --green:#a3be8c; --ver-alpha:#bf616a; --ver-beta:#ebcb8b; --ver-stable:#88c0d0; --syn-cmd:#b5e48c; --syn-bad:#ffb4b4; --syn-flag:#a5f3fc; --syn-str:#fde68a; --syn-path:#bfdbfe; --syn-kw:#f0abfc; --syn-num:#ddd6fe; --syn-op:#f9a8d4; --exec-bg:#242933; }
  body[data-theme="terminal"] { --bg:#000000; --panel:#020a02; --panel2:#010501; --line:#14532d; --text:#4ade80; --dim:#22c55e; --cy:#4ade80; --am:#a3e635; --red:#f87171; --green:#4ade80; --ver-alpha:#f87171; --ver-beta:#a3e635; --ver-stable:#4ade80; --syn-cmd:#4ade80; --syn-bad:#ff8a8a; --syn-flag:#a3e635; --syn-str:#fde047; --syn-path:#2dd4bf; --syn-kw:#bef264; --syn-num:#bbf7d0; --syn-op:#f0abfc; --exec-bg:#000000; }
  /* DHD glyph tuning (build .74): ring and wedges resolve from the theme like the other nav icons; the center stays warm red-orange in the glyph itself. */
  body[data-theme="nord"] .nav-ico-dhd { color:#88c0d0; }
  body[data-theme="terminal"] .nav-ico-dhd { color:#4ade80; }
  body[data-theme="nord"] .nav-ico-mesh { color:#88c0d0; }
  body[data-theme="terminal"] .nav-ico-mesh { color:#4ade80; }
  body[data-theme="glass"] { color-scheme:dark; --bg:#05080d; --panel:rgba(16,26,38,.72); --panel2:rgba(10,17,27,.78); --line:rgba(148,196,220,.35); --text:#f4f8fb; --dim:#c3d0da; --cy:#22d3ee; --am:#f5b544; --red:#ff8a8a; --green:#34d399; --ver-alpha:#ff8a8a; --ver-beta:#f5b544; --ver-stable:#22d3ee; --syn-cmd:#4ade80; --syn-bad:#ff8a8a; --syn-flag:#22d3ee; --syn-str:#fbbf24; --syn-path:#93c5fd; --syn-kw:#f0abfc; --syn-num:#c4b5fd; --syn-op:#e879f9; --exec-bg:#030609; }
  body[data-theme="nocturne"] { color-scheme:dark; --bg:#060a14; --panel:#0b1220; --panel2:#0d1526; --line:#1e3a5f; --text:#dbeafe; --dim:#9db8d9; --cy:#22d3ee; --am:#f471b5; --red:#fb7185; --green:#34d399; --ver-alpha:#fb7185; --ver-beta:#fbbf24; --ver-stable:#22d3ee; --syn-cmd:#22d3ee; --syn-bad:#ff7a93; --syn-flag:#67e8f9; --syn-str:#fbbf24; --syn-path:#93c5fd; --syn-kw:#f471b5; --syn-num:#c4b5fd; --syn-op:#e879f9; --exec-bg:#04070f; }
  body[data-theme="abyss"] { color-scheme:dark; --bg:#010304; --panel:#041014; --panel2:#061a1e; --line:#0f3038; --text:#cfe3e8; --dim:#b8ccd1; --cy:#22e6ee; --am:#67e8f9; --red:#f87171; --green:#2dd4bf; --ver-alpha:#f87171; --ver-beta:#e7c65a; --ver-stable:#22e6ee; --syn-cmd:#67e8f9; --syn-bad:#ff8a8a; --syn-flag:#5eead4; --syn-str:#e7c65a; --syn-path:#cbd5e1; --syn-kw:#7dd3d8; --syn-num:#d9e8ec; --syn-op:#8be9e2; --exec-bg:#000000; }
  body[data-theme="casefile"] { color-scheme:light; --bg:#f3ead8; --panel:#faf4e6; --panel2:#efe4cd; --line:#c9b894; --text:#1a1512; --dim:#5c4a36; --cy:#b91c1c; --am:#92400e; --red:#b91c1c; --green:#166534; --ver-alpha:#b91c1c; --ver-beta:#92400e; --ver-stable:#1d4ed8; --syn-cmd:#86efac; --syn-bad:#fca5a5; --syn-flag:#7dd3fc; --syn-str:#fcd34d; --syn-path:#c4b5fd; --syn-kw:#f9a8d4; --syn-num:#fde68a; --syn-op:#fdba74; --exec-bg:#1a1512; }
  body[data-theme="casefile"] .hlayer { --syn-cmd:#065f46; --syn-bad:#b91c1c; --syn-flag:#075985; --syn-str:#92400e; --syn-path:#3730a3; --syn-kw:#86198f; --syn-num:#5b21b6; --syn-op:#9d174d; }
  body[data-theme="casefile"] .exec, body[data-theme="casefile"] .md-code { font-family:ui-monospace,"Courier New",monospace; }
  body[data-theme="bloodmoon"] { color-scheme:dark; --bg:#070203; --panel:#12050a; --panel2:#180710; --line:#4a1020; --text:#f3e9e4; --dim:#c9a9a0; --cy:#e7c65a; --am:#d4af37; --red:#e11d48; --green:#a3be8c; --ver-alpha:#ff4d6d; --ver-beta:#e7c65a; --ver-stable:#e7c65a; --syn-cmd:#e7c65a; --syn-bad:#ff4d6d; --syn-flag:#93c5fd; --syn-str:#fbbf24; --syn-path:#cbd5e1; --syn-kw:#ff7b54; --syn-num:#fda4af; --syn-op:#ff9e7a; --exec-bg:#0d0205; }
  body[data-theme="phosphor"] { color-scheme:dark; --bg:#050300; --panel:#0d0801; --panel2:#120b02; --line:#7c4a03; --text:#ffd166; --dim:#e8a33d; --cy:#ffb000; --am:#ffd166; --red:#ff6b6b; --green:#ffb000; --ver-alpha:#ff6b6b; --ver-beta:#ffd166; --ver-stable:#ffb000; --syn-cmd:#ffb000; --syn-bad:#ff6b6b; --syn-flag:#ffd166; --syn-str:#fef08a; --syn-path:#ffcf87; --syn-kw:#fb923c; --syn-num:#ffe066; --syn-op:#ff9f1c; --exec-bg:#000000; }
  /* LCARS (v0.6.4e, #324): the palette half — a black field,
     pale blue-white text, and the accent family in the app's
     own slots: ice blue in cyan's, the classic orange in
     amber's, brick in red's, the 2.0 teal in green's. The
     geometry half stands below with the glass theme's
     structural block. */
  body[data-theme="lcars"] { color-scheme:dark; --bg:#000000; --panel:#07070c; --panel2:#0b0b12; --line:#33334a; --text:#dfe8f5; --dim:#9aa8cc; --cy:#99ccff; --blue:#9999ff; --am:#ff9900; --red:#dd4444; --green:#2fd4c4; --ver-alpha:#dd4444; --ver-beta:#ff9900; --ver-stable:#99ccff; --syn-cmd:#99ffcc; --syn-bad:#ff8888; --syn-flag:#99ccff; --syn-str:#ffcc99; --syn-path:#9999ff; --syn-kw:#cc99cc; --syn-num:#ccddff; --syn-op:#ffcc99; --exec-bg:#000000; --lc-orange:#ff9900; --lc-peach:#ffcc99; --lc-lilac:#cc99cc; --lc-peri:#9999ff; --lc-blue:#99ccff; --lc-teal:#33ccdd; --lc-brick:#cc4444; }
  body[data-theme="lcars"] .nav-ico-dhd { color:#99ccff; }
  body[data-theme="lcars"] .nav-ico-mesh { color:#33ccdd; }
  /* Dark Liquid Glass treatment (build .55, #30): resource-light — backdrop blur only on the large surfaces, translucent fill + specular edge on repeated surfaces, multi-glow backdrop, opaque fallbacks. */
  body[data-theme="glass"] { background:#05080d radial-gradient(600px 300px at 15% 0%, rgba(34,211,238,.14), transparent 60%), radial-gradient(700px 400px at 90% 100%, rgba(88,80,236,.12), transparent 60%), radial-gradient(400px 300px at 70% 10%, rgba(52,211,153,.08), transparent 60%); background-attachment:fixed; }
  body[data-theme="glass"] nav, body[data-theme="glass"] aside { background:rgba(10,17,27,.55); backdrop-filter:blur(12px) saturate(1.4); -webkit-backdrop-filter:blur(12px) saturate(1.4); }
  body[data-theme="glass"] .card, body[data-theme="glass"] .msg, body[data-theme="glass"] .foldcard { background:rgba(255,255,255,.05); box-shadow:inset 0 1px 0 rgba(255,255,255,.18), 0 4px 18px rgba(0,0,0,.35); border-color:rgba(148,196,220,.35); }
  body[data-theme="glass"] .ac, body[data-theme="glass"] .acsearch { background:rgba(10,17,27,.82); backdrop-filter:blur(12px) saturate(1.4); -webkit-backdrop-filter:blur(12px) saturate(1.4); }
  @supports not ((backdrop-filter:blur(4px)) or (-webkit-backdrop-filter:blur(4px))) { body[data-theme="glass"] nav, body[data-theme="glass"] aside, body[data-theme="glass"] .ac, body[data-theme="glass"] .acsearch { background:#0a111b; backdrop-filter:none; -webkit-backdrop-filter:none; } body[data-theme="glass"] .card, body[data-theme="glass"] .msg { background:#101a26; } }
  @media (prefers-reduced-transparency: reduce) { body[data-theme="glass"] nav, body[data-theme="glass"] aside, body[data-theme="glass"] .ac, body[data-theme="glass"] .acsearch { background:#0a111b; backdrop-filter:none; -webkit-backdrop-filter:none; } body[data-theme="glass"] .card, body[data-theme="glass"] .msg, body[data-theme="glass"] .foldcard { background:#101a26; } }
  /* LCARS geometry (v0.6.4e, #324): skin, not layout — the
     same DOM under a theme class, per the addendum's lock.
     The reference is LCARS 2.0 (Alex's amendment): sleeker
     proportions than the 1987 original — thinner bars and
     pills, cooler blue/teal woven through the classic
     palette, denser readouts, more black space. The grammar:
     pill-ended bars and buttons with BLACK text on color,
     elbow curves on cards and panels, uppercase condensed
     micro-labels on the furniture (never on authored
     content — chat, notes and documents read as written).
     Typography is local-first: the system condensed stack,
     no fetched font. The rail's collapsible groups keep
     their exact mechanics — a collapsed group is its header
     pill alone; nothing here touches the collapse state,
     the grid, or the aria. */
  body[data-theme="lcars"] { font-family:"Arial Narrow","Liberation Sans Narrow","Roboto Condensed","Helvetica Neue Condensed","Helvetica Neue",Arial,sans-serif; }
  body[data-theme="lcars"] nav, body[data-theme="lcars"] aside { background:#000000; }
  body[data-theme="lcars"] h1, body[data-theme="lcars"] h2, body[data-theme="lcars"] h3, body[data-theme="lcars"] h4 { text-transform:uppercase; letter-spacing:.14em; color:var(--lc-peach); font-weight:600; }
  body[data-theme="lcars"] h3 { border-bottom:3px solid var(--lc-orange); padding-bottom:5px; }
  body[data-theme="lcars"] button { border-radius:999px; text-transform:uppercase; letter-spacing:.09em; font-weight:600; }
  body[data-theme="lcars"] button.act { background:var(--lc-orange); color:#000000; border-color:transparent; }
  body[data-theme="lcars"] button.act:hover { background:var(--lc-peach); }
  body[data-theme="lcars"] button.ghost { background:transparent; border:1.5px solid var(--lc-peri); color:var(--lc-blue); }
  body[data-theme="lcars"] button.ghost:hover { border-color:var(--lc-blue); color:#000000; background:var(--lc-blue); }
  body[data-theme="lcars"] .card, body[data-theme="lcars"] .foldcard { background:#050508; border:1px solid #1c1c28; border-left:8px solid var(--lc-orange); border-radius:22px 6px 6px 22px; }
  body[data-theme="lcars"] .card > b { text-transform:uppercase; letter-spacing:.1em; color:var(--lc-blue); }
  body[data-theme="lcars"] .msg { border-radius:18px 6px 6px 18px; border-left:6px solid var(--lc-blue); background:#050508; }
  body[data-theme="lcars"] input, body[data-theme="lcars"] select, body[data-theme="lcars"] textarea { border-radius:999px; border:1.5px solid var(--lc-peri); background:#000000; color:var(--text); }
  body[data-theme="lcars"] textarea { border-radius:14px; }
  body[data-theme="lcars"] .switch { background:#1a1a26; border:1.5px solid var(--lc-lilac); }
  body[data-theme="lcars"] .switch .knob { background:var(--lc-lilac); }
  body[data-theme="lcars"] .switch[aria-checked="true"] { background:var(--lc-orange); border-color:var(--lc-orange); }
  body[data-theme="lcars"] .switch[aria-checked="true"] .knob { background:#000000; }
  body[data-theme="lcars"] #nav-menu button { border-radius:999px; text-transform:uppercase; letter-spacing:.08em; }
  body[data-theme="lcars"] .nav-group-head { background:var(--lc-peri); color:#000000; border-radius:999px; }
  body[data-theme="lcars"] .nav-group-head .nav-chev { color:#000000; }
  body[data-theme="lcars"] #nav-group-network { background:var(--lc-blue); }
  body[data-theme="lcars"] #nav-group-navi { background:var(--lc-orange); }
  body[data-theme="lcars"] #nav-group-apps { background:var(--lc-lilac); }
  body[data-theme="lcars"] #nav-group-utilities { background:var(--lc-teal); }
  body[data-theme="lcars"] #nav-menu .nav-child { border:1.5px solid #2c2c40; color:var(--text); background:#050508; }
  body[data-theme="lcars"] #nav-menu .nav-child.on { background:var(--lc-peach); color:#000000; border-color:transparent; }
  body[data-theme="lcars"] .nav-break { border-top:3px solid var(--lc-brick); }
  body[data-theme="lcars"] ::-webkit-scrollbar { width:8px; height:8px; }
  body[data-theme="lcars"] ::-webkit-scrollbar-thumb { background:var(--lc-peri); border-radius:999px; }
  body[data-theme="lcars"] ::-webkit-scrollbar-track { background:#000000; }
  /* LCARS Mode (v0.6.5e, #325): the Layout axis. Where the LCARS
     theme above is a palette, this block is GEOMETRY — keyed on
     <body data-layout="lcars">, independent of data-theme, so
     every one of the twelve palettes wears it. The accent slots
     are read from the ACTIVE theme (--am, --cy, --blue, --green,
     --red); under the LCARS palette itself the classic --lc-*
     family takes those slots. Furniture only — headings, labels,
     the rail, buttons, cards: authored content (chat text, case
     text, document text) is never transformed. The mode owns no
     colors of its own beyond the grammar (black text on a solid
     accent, as the theme's skin already established). */
  body[data-layout="lcars"] { --lay-1:var(--am); --lay-2:var(--cy); --lay-3:var(--blue); --lay-4:var(--green); --lay-5:var(--red); }
  body[data-layout="lcars"][data-theme="lcars"] { --lay-1:var(--lc-orange); --lay-2:var(--lc-blue); --lay-3:var(--lc-peri); --lay-4:var(--lc-teal); --lay-5:var(--lc-brick); }
  body[data-layout="lcars"] { font-family:"Arial Narrow","Liberation Sans Narrow","Roboto Condensed","Helvetica Neue Condensed","Helvetica Neue",Arial,sans-serif; }
  body[data-layout="lcars"] h1, body[data-layout="lcars"] h2, body[data-layout="lcars"] h3, body[data-layout="lcars"] h4 { text-transform:uppercase; letter-spacing:.13em; font-weight:600; }
  body[data-layout="lcars"] label { text-transform:uppercase; letter-spacing:.08em; }
  body[data-layout="lcars"] nav { border-right:6px solid var(--lay-1); counter-reset:laygrp; }
  body[data-layout="lcars"] aside { border-left:6px solid var(--lay-3); }
  body[data-layout="lcars"] .nav-group-head { background:var(--lay-3); color:#000000; border-radius:999px; text-transform:uppercase; letter-spacing:.08em; font-weight:600; counter-increment:laygrp; }
  body[data-layout="lcars"] .nav-group-head::before { content:counter(laygrp, decimal-leading-zero); font-weight:700; margin-right:7px; opacity:.7; }
  body[data-layout="lcars"] .nav-group-head .nav-chev { color:#000000; }
  body[data-layout="lcars"] #nav-group-network { background:var(--lay-2); }
  body[data-layout="lcars"] #nav-group-navi { background:var(--lay-1); }
  body[data-layout="lcars"] #nav-group-apps { background:var(--lay-3); }
  body[data-layout="lcars"] #nav-group-utilities { background:var(--lay-4); }
  body[data-layout="lcars"] #nav-group-systems { background:var(--lay-2); }
  body[data-layout="lcars"] #nav-menu button { border-radius:999px; text-transform:uppercase; letter-spacing:.07em; }
  body[data-layout="lcars"] #nav-menu .nav-child { border:1.5px solid var(--line); }
  body[data-layout="lcars"] #nav-menu .nav-child.on { background:var(--lay-1); color:#000000; border-color:transparent; }
  body[data-layout="lcars"] .nav-break { border-top:3px solid var(--lay-5); }
  body[data-layout="lcars"] #view-root > h3 { background:color-mix(in srgb, var(--lay-1) 24%, transparent); border-left:14px solid var(--lay-1); border-radius:999px; padding:5px 14px; }
  body[data-layout="lcars"] .card, body[data-layout="lcars"] .foldcard { border-left:10px solid var(--lay-1); border-radius:26px 8px 8px 26px; }
  body[data-layout="lcars"] .card > h3, body[data-layout="lcars"] .foldcard > h3 { border-bottom:3px solid var(--lay-1); padding-bottom:5px; }
  body[data-layout="lcars"] .card > b { text-transform:uppercase; letter-spacing:.1em; }
  body[data-layout="lcars"] .msg { border-radius:18px 6px 6px 18px; border-left:6px solid var(--lay-2); }
  body[data-layout="lcars"] button { border-radius:999px; text-transform:uppercase; letter-spacing:.09em; font-weight:600; }
  body[data-layout="lcars"] button.act { background:var(--lay-1); color:#000000; border-color:transparent; }
  body[data-layout="lcars"] button.ghost { border:1.5px solid var(--lay-3); }
  body[data-layout="lcars"] input, body[data-layout="lcars"] select, body[data-layout="lcars"] textarea { border-radius:999px; border:1.5px solid var(--lay-3); }
  body[data-layout="lcars"] textarea { border-radius:14px; }
  body[data-layout="lcars"] .switch { border:1.5px solid var(--lay-3); }
  body[data-layout="lcars"] .switch[aria-checked="true"] { background:var(--lay-1); border-color:var(--lay-1); }
  body[data-layout="lcars"] .switch[aria-checked="true"] .knob { background:#000000; }
  body[data-layout="lcars"] ::-webkit-scrollbar { width:8px; height:8px; }
  body[data-layout="lcars"] ::-webkit-scrollbar-thumb { background:var(--lay-3); border-radius:999px; }
  /* LCARS Mode on the dedicated mobile interface (<=800px): the
     same geometry at phone width — slimmer elbow, full-size
     touch targets on the rail and the fields, and the view
     never scrolls sideways. */
  @media (max-width:800px) {
    body[data-layout="lcars"] .card, body[data-layout="lcars"] .foldcard { border-left-width:8px; border-radius:20px 6px 6px 20px; }
    body[data-layout="lcars"] #nav-menu button, body[data-layout="lcars"] .nav-group-head { min-height:44px; }
    body[data-layout="lcars"] input, body[data-layout="lcars"] select { min-height:44px; }
    body[data-layout="lcars"] #view-root { overflow-x:hidden; }
  }
  /* Nordic icon color-coding: each menu icon carries its own color
     from the Aurora/Frost family — the same color in the expanded
     menu and the collapsed rail, since the icon span never changes.
     Every color clears 3:1 on the Polar Night surfaces. The active
     item trades the Frost fill for the deepest Polar Night step
     with a Frost edge, so its icon's own color stays readable. */
  body[data-theme="nord"] #nav-Chat .ic { color:#88c0d0; }
  body[data-theme="nord"] #nav-child-navi-Chips .ic { color:#ebcb8b; }
  body[data-theme="nord"] #nav-child-navi-RAG .ic { color:#81a1c1; }
  body[data-theme="nord"] #nav-Vault .ic { color:#bf616a; }
  body[data-theme="nord"] #nav-child-utilities-GitHub .ic { color:#d08770; }
  body[data-theme="nord"] #nav-child-utilities-Settings .ic { color:#d8dee9; }
  body[data-theme="nord"] #nav-child-utilities-Docs .ic { color:#d98a94; }
  body[data-theme="nord"] #nav-child-network-FediChat .ic { color:#a3be8c; }
  body[data-theme="nord"] [id="nav-child-apps-The Dossier"] .ic { color:#b48ead; }
  body[data-theme="nord"] nav button.on, body[data-theme="nord"] nav button.on:hover { background:#2e3440; color:#eceff4; box-shadow:inset 2px 0 0 #88c0d0; }
  body[data-theme="nord"] nav button:hover { background:rgba(136,192,208,.14); }
  body[data-theme="nord"] .card:hover { border-color:#5e81ac; }
  /* Nordic markdown: her rendered answers follow the palette —
     Snow Storm words (the body text already inherits them), Frost
     headings and links, inline code on a Frost wash, code blocks
     and terminal output on the deeper Polar Night step with Nord
     syntax colors, quotes ruled in muted Frost. The wiki's section
     heads and the editor's surfaces and tokens follow too. */
  body[data-theme="nord"] .md-h { color:#88c0d0; }
  body[data-theme="nord"] .md .ic { background:rgba(136,192,208,.16); color:#88c0d0; }
  body[data-theme="nord"] .md-code { background:#242933; color:#d8dee9; border-color:#4c566a; }
  body[data-theme="nord"] .md-q { border-color:#5e81ac; }
  body[data-theme="nord"] .exec { background:#242933; color:#d8dee9; }
  body[data-theme="nord"] pre.out { background:#242933; }
  body[data-theme="nord"] .doc-sec h4 { color:#88c0d0; }
  body[data-theme="nord"] .ac-it.on { background:rgba(136,192,208,.16); }
  body[data-theme="nord"] .tok-com { color:#616e88; }
  body[data-theme="nord"] .tok-str { color:#a3be8c; }
  body[data-theme="nord"] .tok-kw { color:#81a1c1; }
  body[data-theme="nord"] .tok-num { color:#b48ead; }
  body[data-theme="nord"] .tok-dec { color:#d08770; }
  body[data-theme="nord"] .tok-navi { color:#88c0d0; }
  body[data-theme="nord"] .tok-call { color:#8fbcbb; }
  body[data-theme="nord"] .ed-wrap { background:#242933; }
  body[data-theme="nord"] .ed-gutter { background:#2e3440; }
  body[data-theme="nord"] .ed-mini { background:#2e3440; }
  body[data-theme="nord"] .ed-extras i { background:#b48ead; }
  body[data-theme="terminal"] #nav-Chat .ic { color:#4ade80; }
  body[data-theme="terminal"] #nav-child-navi-Chips .ic { color:#34d399; }
  body[data-theme="terminal"] #nav-child-navi-RAG .ic { color:#2dd4bf; }
  body[data-theme="terminal"] #nav-Vault .ic { color:#fde047; }
  body[data-theme="terminal"] #nav-child-utilities-GitHub .ic { color:#facc15; }
  body[data-theme="terminal"] #nav-child-utilities-Settings .ic { color:#dcfce7; }
  body[data-theme="terminal"] #nav-child-utilities-Docs .ic { color:#6ee7b7; }
  body[data-theme="terminal"] nav button.on, body[data-theme="terminal"] nav button.on:hover { background:#052e16; color:#dcfce7; box-shadow:inset 2px 0 0 #4ade80; }
  body[data-theme="terminal"] .md-h { color:#a3e635; }
  body[data-theme="terminal"] .md-code { background:#000000; color:#86efac; border-color:#14532d; }
  body[data-theme="terminal"] .exec { background:#000000; color:#86efac; }
  * { box-sizing:border-box; }
  html, body { height:100%; }
  body { margin:0; background:var(--bg); color:var(--text); font:14px/1.5 system-ui,sans-serif; overflow:hidden; }
  .mono { font-family:ui-monospace,Menlo,Consolas,monospace; }
  /* Motion tokens (0.1.9e Build 6, issue #109): ONE motion
     system for the whole app — every transition and animation
     below takes its duration from these variables and its
     easing from the two curves, so the app moves in one
     language: fast for hovers and presses, med for unfolds,
     slides, and pane swaps, slow for the one big flip, view
     for view entrances. Under prefers-reduced-motion the
     tokens themselves collapse to zero — the operating
     system's word always wins over the Potato toggle in
     Settings -> Appearance, which applies the same collapse
     through body[data-motion="potato"] at the foot of this
     stylesheet. */
  :root { --motion-fast:.14s; --motion-med:.3s; --motion-slow:.55s; --motion-view:.22s; --motion-glide:.28s; --motion-press:.08s; --motion-ease:cubic-bezier(.22,.8,.3,1); --uiscale:1; }
  @media (prefers-reduced-motion: reduce) { :root { --motion-fast:0s; --motion-med:0s; --motion-slow:0s; --motion-view:0s; --motion-glide:0s; --motion-press:0s; } }
  /* Fixed-frame dashboard: the app fills the viewport and the page
     itself never scrolls. The left menu and the right status column
     stay fixed in place; only the center column scrolls — and in the
     Chat view only the message thread scrolls, with the composer
     pinned beneath it. */
  /* Build 6 (#111): the expanded menu is 196px — at 170px the
     group headers' labels + auto-margin chevron overflowed the
     rail and the chevron clipped at its edge. The frame height
     divides by --uiscale: the UI scale (body zoom) magnifies
     used lengths, so the frame asks for 100dvh / scale and
     lands exactly on the viewport at every scale. */
  #app { display:grid; grid-template-columns:196px minmax(0,1fr) 230px; grid-template-rows:minmax(0,1fr); height:calc(100vh / var(--uiscale, 1)); height:calc(100dvh / var(--uiscale, 1)); overflow:hidden; }
  #app.closed { grid-template-columns:72px minmax(0,1fr) 230px; }
  /* The status column starts hidden (Build 6, #114, as
     corrected): on desktop widths the column's 230px goes to
     the content until the operator opens it with the pane
     button in her presence bar. Scoped to desktop — on phones
     the column is the dock overlay, whose semantics are
     unchanged and never gated by this class. The class is in
     the static markup, so the first paint is already the
     two-column frame; the grid transition animates only the
     operator's toggles. */
  @media (min-width:801px) {
    #app.no-status { grid-template-columns:196px minmax(0,1fr) 0px; }
    #app.no-status.closed { grid-template-columns:72px minmax(0,1fr) 0px; }
    #app.no-status aside { display:none; }
  }
  nav { border-right:1px solid var(--line); background:var(--panel2); padding:14px 10px; height:100%; min-height:0; overflow-y:auto; overflow-x:hidden; display:flex; flex-direction:column; }
  /* Her version at the foot of the menu, under a divider bar: the
     number is read from the VERSION file at the install root (the
     bridge reports it on /api/status and CONFIG.version follows
     it), so the menu always shows the file's truth. It reads in
     GitHub tag form - v + number + status tag ("v0.3.0e") -
     centered in the footer's row, and the collapsed rail keeps
     the same whole string, centered. (v0.5.2e, #255: at 54px
     the string did NOT fit — the field showed it clipped —
     so the collapsed rail is 72px now, the operator's own
     latitude, and the stamp sits centered and whole.) */
  /* Build .87 (Alex's ruling): Docs and Settings leave the main
     stack and sit at the foot of the menu, stacked directly over
     the version line. Superseded in 0.1.9e Build 4: the foot is
     the Utilities group now — its container (.nav-util)
     carries the auto margin, its children (Settings, Docs,
     GitHub) open UPWARD over the header, and the version follows
     the group with no gap. */
  .nav-util { margin-top:auto; display:flex; flex-direction:column; }
  #nav-version { border-top:1px solid var(--line); padding:9px 8px 0; color:var(--dim); font-size:11px; white-space:nowrap; text-align:center; }
  #nav-version .nv-num { color:var(--text); font-weight:700; }
  #nav-version .nv-val { font-weight:700; }
  #nav-version.st-alpha .nv-val, #nav-version.st-edge .nv-val { color:var(--ver-alpha); }
  #nav-version.st-beta .nv-val { color:var(--ver-beta); }
  #nav-version.st-stable .nv-val { color:var(--ver-stable); }
  #app.closed #nav-version { text-align:center; padding-left:0; padding-right:0; }
  nav .logo { color:var(--cy); font-weight:700; letter-spacing:.18em; padding:4px 8px 2px; white-space:nowrap; }
  /* Her tagline under the wordmark: one static line — "Powered by
     Navi" — standing still where the rotating loglines used to
     cycle. Hidden when the menu collapses to its rail. */
  nav .tagline { color:var(--dim); font-size:11px; letter-spacing:.04em; line-height:1.45; padding:0 8px 12px; }
  #app.closed nav .tagline { display:none; }
  @keyframes navi-glitch { 0% { opacity:1; transform:translate(0); text-shadow:none; clip-path:inset(0 0 0 0); } 15% { opacity:.7; transform:translate(-2px,1px) skewX(-4deg); text-shadow:2px 0 rgba(248,113,113,.8), -2px 0 rgba(34,211,238,.8); clip-path:inset(12% 0 55% 0); } 30% { opacity:.9; transform:translate(2px,-1px); text-shadow:-3px 0 rgba(248,113,113,.8), 3px 0 rgba(34,211,238,.8); clip-path:inset(60% 0 8% 0); } 45% { opacity:.4; transform:translate(-1px,0) skewX(3deg); clip-path:inset(30% 0 30% 0); } 60% { opacity:.8; transform:translate(3px,1px); text-shadow:2px 0 rgba(34,211,238,.8), -2px 0 rgba(248,113,113,.8); clip-path:inset(5% 0 78% 0); } 80% { opacity:.6; transform:translate(-2px,0); clip-path:inset(45% 0 40% 0); } 100% { opacity:1; transform:translate(0); text-shadow:none; clip-path:inset(0 0 0 0); } }
  .glitching { animation:navi-glitch .45s steps(2,end) both; }
  /* Minimized, the banner stays exactly as it is — the full
     "APEX", never clipped: it simply wraps inside the rail at
     a smaller size instead of losing part of the word. */
  #app.closed nav .logo { white-space:normal; overflow:visible; letter-spacing:.05em; font-size:10px; line-height:1.4; text-align:center; padding:4px 0 12px; }
  nav button { display:flex; align-items:center; gap:8px; width:100%; text-align:left; background:none; border:0; color:var(--dim); padding:8px; border-radius:4px; cursor:pointer; font:inherit; white-space:nowrap; overflow:hidden; }
  nav button.on { background:var(--cy); color:#000; font-weight:700; }
  nav button .ic { display:inline-flex; flex:0 0 auto; }
  nav button .ic svg { display:block; }
  #app.closed nav button { justify-content:center; }
/* v0.6.0e (#292): in the EXPANDED menu the Chat entry reads
   as a peer of the centered Home tile — its icon and label
   centered as one unit, the way the collapsed rail already
   centers it. Section rows (headers and chevroned doors)
   keep their left-aligned grammar untouched. */
#nav-Chat { justify-content:center; }
  #app.closed nav button .lbl { display:none; }
  /* Minimized to the rail, hovering (or tabbing to) an icon names
     the view in a tooltip beside it; the expanded menu shows its
     labels and the tooltip span stays hidden. The closed nav and
     its buttons let the tooltip paint outside the rail. */
  .nav-tip { position:absolute; left:calc(100% + 10px); top:50%; transform:translateY(-50%); background:var(--panel); color:var(--text); border:1px solid var(--line); padding:3px 8px; font-size:11px; border-radius:4px; white-space:nowrap; opacity:0; pointer-events:none; transition:opacity var(--motion-fast) ease; z-index:60; display:none; }
  #app.closed nav { overflow:visible; }
  #app.closed nav button { overflow:visible; position:relative; }
  #app.closed .nav-tip { display:block; }
  #app.closed nav button:hover .nav-tip, #app.closed nav button:focus-visible .nav-tip { opacity:1; }
  @media (prefers-reduced-motion: reduce) { .nav-tip { transition:none; } }
  /* Grouped menu + motion (0.1.9e Build 4; Alex: "beautifully
     animated and fluid — lean into the web tech"). The language:
     groups open and close on the grid-template-rows technique
     (the same one the Docs topics use) — a ~300ms unfold on a
     soft cubic-bezier settle, chosen for feel (Alex: beauty has
     the floor on these small surfaces) — the
     chevron turns with the state, and children arrive with a
     slight stagger — each child's transition-delay is set from
     its index when the menu is built, and only transform/opacity
     animate, so the surfaces stay on the GPU path and hold 60fps
     on modest hardware. No backdrop filters on animated elements,
     no idle animation loops: transitions fire on state changes
     only. The Utilities group at the foot opens upward — its
     children stack ABOVE the header in the DOM, so the same
     height motion reads as a drop-up. View switches keep the
     standing .22s view-in fade (below). Under
     prefers-reduced-motion every one of these collapses to an
     instant state change (the global rule at the foot of this
     block and the standing reduced-motion rules). */
  .nav-break { border-top:1px solid var(--line); margin:8px 4px; flex:0 0 auto; }
  #app.closed .nav-break { margin:8px 2px; }
  .nav-group-head { font-weight:700; color:var(--text); }
  .nav-group-head .nav-chev { margin-left:auto; font-size:11px; color:var(--dim); transition:transform var(--motion-med) var(--motion-ease); }
  .nav-group-head[aria-expanded="false"] .nav-chev { transform:rotate(-90deg); }
  #app.closed .nav-group-head .nav-chev, #app.closed .nav-group-head .lbl { display:none; }
  #app.closed .nav-group-head { justify-content:center; }
  .nav-children { display:grid; grid-template-rows:1fr; transition:grid-template-rows var(--motion-med) var(--motion-ease); min-height:0; }
  .nav-children.closed { grid-template-rows:0fr; }
  /* Build 6 (#105): the children ride in ONE inner track, the
     shape the Docs topics always used. Build 4 made each button
     a direct grid child, so grid-template-rows templated only
     the FIRST row and the remaining children overflowed the
     track onto the rows after the group (Mesh over Voice, DHD
     over Chips on phones). One track, one templated row: the
     unfold animates the track, and nothing can paint outside
     it. The inner clips while the track is mid-flight, and at
     0fr the wrap is exactly zero tall — a collapsed group
     leaves no void in the flow on any screen size. */
  .nav-children-inner { overflow:hidden; min-height:0; display:flex; flex-direction:column; }
  .nav-children-inner > button { min-height:0; overflow:hidden; min-width:0; }
  .nav-group-sec { display:flex; flex-direction:column; }
  .nav-children .nav-child { opacity:1; transform:none; transition:opacity var(--motion-glide) var(--motion-ease), transform var(--motion-glide) var(--motion-ease), background var(--motion-fast) ease, color var(--motion-fast) ease; }
  .nav-children.closed .nav-child { opacity:0; transform:translateY(-4px); pointer-events:none; }
  #nav-children-utilities .nav-child { transform:none; }
  .nav-children.closed#nav-children-utilities .nav-child { transform:translateY(4px); }
  nav button { transition:background var(--motion-fast) ease, color var(--motion-fast) ease; }
  /* The tabbed status pane (0.1.9e Build 6, issues #107/#108):
     the status column is two panes under one sticky tab bar —
     Status first, Chats second. Build 4's Chats disclosure is
     retired: its panel opened at the BOTTOM of the column, a
     full scroll away from its tab (#107). Now exactly one pane
     shows, swapped in place directly under the tab bar, and
     the bar sticks to the top of the column while it scrolls.
     While Chats is up, an amber dot on the Status tab carries
     any summons (update waiting, stale bridge or page) so a
     tab can never bury one. */
  /* The Chat window's own header (Build 6, #115): her presence
     at the top left — name, dot, and the living line, social-
     chat style — and the status column's toggle hugging the
     right edge, a full 44px target in the panel-icon language. */
  /* Her presence bar (Build 8): frame chrome above every view —
     her portrait, dot, name and living line, with the status
     column's pane button hugging the right edge. One persistent
     element, mounted outside the per-view render. The pane
     button's sizing rule is its pre-#123 one (Build 6), restored
     with the button: a 44px target whose pressed state wears
     the accent — keyed to the frame's own state classes since
     Build 9 (#130), the classes that show the column/dock. */
  #presence-bar { display:flex; align-items:center; gap:10px; flex:0 0 auto; margin-bottom:10px; }
  #chat-status-toggle { flex:0 0 auto; width:44px; height:44px; min-width:44px; min-height:44px; display:inline-flex; align-items:center; justify-content:center; padding:0; transition:background var(--motion-fast) ease, filter var(--motion-fast) ease, transform var(--motion-press) ease; }
  @media (max-width:800px) { #app.dock-open #chat-status-toggle { color:var(--cy); border-color:var(--cy); } }
  @media (min-width:801px) { #app:not(.no-status) #chat-status-toggle { color:var(--cy); border-color:var(--cy); } }
  #presence-bar .chat-presence { display:flex; align-items:center; gap:9px; flex:1 1 auto; min-width:0; }
  .chat-presence .dot { flex:0 0 auto; }
  .chat-presence-name { font-weight:700; line-height:1.25; }
  .chat-presence-line { color:var(--dim); font-size:12px; line-height:1.3; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  /* The status pane's presence row (0.1.9e Build 11, #140)
     is retired in v0.4.10e (#250): #248 took the presence
     line out of the pane, and what remained — a STATUS header
     over a lone green dot on the wide canvas — was an orphan,
     not a section. The pane now opens on its first real
     section; her portrait, name, and dot live at the top of
     the chat, untouched. */
  .status-tabs { position:sticky; top:-14px; z-index:5; display:flex; gap:6px; padding:12px 0 8px; margin:-14px 0 0; background:var(--panel2); border-bottom:1px solid var(--line); }
  .status-tab { flex:1 1 0; display:flex; align-items:center; justify-content:center; gap:7px; min-height:44px; padding:8px 10px; background:var(--panel); border:1px solid var(--line); border-radius:6px; color:var(--dim); font:inherit; font-weight:700; cursor:pointer; transition:background var(--motion-fast) ease, color var(--motion-fast) ease, border-color var(--motion-fast) ease; }
  .status-tab:hover { border-color:var(--cy); color:var(--text); }
  .status-tab.on { color:var(--text); border-color:var(--cy); }
  .status-tab .chats-count { color:var(--dim); font-size:11px; }
  .status-attn { width:8px; height:8px; border-radius:50%; background:#f5b544; flex:0 0 auto; }
  .status-pane-in { animation:view-in var(--motion-view) ease both; }
  @media (prefers-reduced-motion: reduce) { .nav-children, .nav-group-head .nav-chev, .nav-children .nav-child { transition:none; } .status-pane-in { animation:none; } }
  #nav-toggle { color:var(--dim); border:1px solid var(--line); margin-bottom:8px; }
  /* The destination buttons and the version footer live in one
     container so the small-screen rules can lift them out of the
     top bar as a single overlay panel; on desktop the container is
     display:contents, so the nav's flex column sees exactly the
     children it always had. The dock toggle is phone chrome only:
     hidden here, shown by the small-screen block. */
  #nav-menu { display:contents; }
  #dock-toggle { display:none; }
  /* The phone top bar's center line — "Apex | powered by Navi" on
     one line between the hamburger and the dock toggle. Phone
     chrome only: the small-screen block shows it and puts the
     stacked wordmark/tagline away; desktop never renders it. */
  .bar-title { display:none; }
  main { padding:16px; min-width:0; height:100%; min-height:0; overflow-y:auto; display:flex; flex-direction:column; }
  /* The view container (Build 8): render() writes views here,
     below the persistent presence bar, so the frame classes
     (chat-view / docs-view / docs-pop / view-in) land on it —
     it carries the sizing and scrolling role main had alone
     before the bar existed. */
  #view-root { flex:1 1 auto; min-height:0; overflow-y:auto; }
  /* Card squash guard (build .86, issue #79): a card in a scroll
     column keeps its full content height — flex:0 0 auto, so a
     Local chip card (or any card list with the same pattern) is
     never sliced mid-text by a flex parent again. */
  main .card { flex:0 0 auto; }
  #view-root.chat-view { display:flex; flex-direction:column; overflow:hidden; }
  /* The frame reaches the window's foot (0.1.9e Build 11,
     issue #133): the composer carries a negative bottom
     margin — it bleeds over main's bottom padding so its
     panel meets the window's edge — but this frame's
     overflow:hidden clipped the composer at the frame's own
     bottom edge, which sits one padding (16px) above the
     window's: the hint row's bottom band — the mode pill's
     and the chat-only button's lower borders — was sliced
     at rest, empty thread or full. The frame now takes a
     negative bottom margin of the same measure, so its
     bottom edge IS the window's bottom edge and the
     composer's bleed lands inside the frame; the composer
     additionally gives up its own negative bottom margin
     inside the frame, so its margin box is its true height
     and the thread yields exactly that. Every painted box
     keeps the place it had — the band that was clipped is
     simply painted now. The phone block below restates the
     margin at the phone padding (12px). */
  #view-root.chat-view { margin-bottom:-16px; }
  #view-root.chat-view > .composer { margin-bottom:0; }
  /* The Docs view's own layout frame (issue #119): the view is
     a fixed frame — header still, and below it two panes that
     scroll independently, the topic menu and the article, each
     filling the frame's height exactly. No page-level scroll. */
  #view-root.docs-view { display:flex; flex-direction:column; overflow:hidden; }
  #view-root.docs-view > h3 { flex:0 0 auto; }
  .docs-frame { display:flex; flex-direction:column; flex:1; min-height:0; }
  .docs-panes { display:flex; gap:16px; flex:1; min-height:0; margin-top:10px; }
  .docs-wiki-nav { min-width:190px; overflow-y:auto; overscroll-behavior:contain; scrollbar-gutter:stable; min-height:0; }
  .docs-wiki-body { flex:1; min-width:260px; overflow-y:auto; overscroll-behavior:contain; min-height:0; }
  #view-root.docs-pop { height:100dvh; display:flex; flex-direction:column; overflow:hidden; }
  /* The Wire's article reader (0.2.0e Build 10, #196): while it
     is open the reader is portaled to <body> — outside the view
     root's scroll and animation ancestry — so its fixed
     positioning anchors to the viewport on every door. Desktop:
     the centered popup, as it always was. Phone widths: a
     full-screen sheet above its scrim, its own scroll inside,
     the page behind it locked.
     Build 11 (#196 rider): the panel no longer scrolls as one
     box with a sticky first child. The field screenshot showed
     the headline's first line half-clipped at the sheet's top
     edge, above the source/Close bar — the bar and the headline
     shared one scroll box, so the bar's painted place at open
     rode on the engine's sticky-vs-padding origin and the
     panel's initial scroll origin. Now the panel is a flex
     column that never scrolls: the bar is a fixed first child
     and the article body scrolls in its own box below it, so
     at open the headline stands fully below the bar and at no
     scroll position can the bar overlap or clip a content
     line — on either door, by construction. The desktop panel
     also widens (680px was a newspaper column; the operator
     asked for the page). */
  #wire-reader-shade { position:fixed; inset:0; z-index:118; background:rgba(0,0,0,.45); }
  #wire-reader { position:fixed; z-index:119; left:50%; top:50%; transform:translate(-50%,-50%); width:min(1120px,calc(100vw - 48px)); max-height:86vh; overflow:hidden; display:flex; flex-direction:column; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:0; transition:opacity var(--motion-med) var(--motion-ease),transform var(--motion-med) var(--motion-ease); }
  .wire-reader-head { flex:0 0 auto; background:var(--panel2); padding:20px 20px 8px; }
  .wire-reader-body { flex:1 1 auto; min-height:0; overflow-y:auto; padding:0 20px 18px; }
  @media (max-width:800px) {
    #wire-reader { left:0; top:0; right:0; bottom:0; transform:none; width:auto; max-height:none; height:100dvh; border:none; border-radius:0; box-shadow:none; padding:0; }
    .wire-reader-head { padding:16px 16px 8px; }
    .wire-reader-body { padding:0 16px calc(14px + env(safe-area-inset-bottom)); }
  }
  body.reader-open { overflow:hidden; }
  #view-root.chat-view > h3, #view-root.chat-view > .card { flex:0 0 auto; }
  #msgs { flex:1 1 auto; min-height:0; overflow-y:auto; padding-right:4px; }
  #msgs + div, #view-root.chat-view > .row { flex:0 0 auto; }
  #view-root.chat-view > .row { padding-top:8px; }
  aside { border-left:1px solid var(--line); background:var(--panel2); padding:14px; height:100%; max-height:100%; min-height:0; overflow-y:auto; overscroll-behavior:contain; scrollbar-gutter:stable; }
  aside h3, main h3 { color:var(--dim); font-size:11px; letter-spacing:.18em; text-transform:uppercase; margin:0 0 8px; }
  .stat { margin-bottom:18px; padding-bottom:16px; border-bottom:1px solid var(--line); } .stat .v { color:var(--text); }
  .stat:last-child { border-bottom:none; padding-bottom:0; margin-bottom:2px; }
  .dot { display:inline-block; width:8px; height:8px; border-radius:50%; margin-right:6px; }
  .card { background:var(--panel); border:1px solid var(--line); border-radius:6px; padding:12px; margin-bottom:12px; }
  /* The Vault pair (build .80, issue #68; placement reversed
     in 0.1.9e Build 11, issue #134, superseding #68): an
     explicit two-column grid spanning the full width of the
     view — the files panel assigned to column 1, the editor
     to column 2 — replacing the reversed flex row (.73) whose
     rendered placement diverged between the shell webview
     and browsers at wide widths. (#68's grid put the files
     panel on the RIGHT; the operator's 2026-11-11 direction
     reverses it — the tree leads, the editor follows, the
     way the app's own rail leads its views.) Grid column
     assignment renders identically in every engine, and
     width:100% anchors the pair to the view's edges, never
     mid-page. The phone block below restacks it. */
  .vault-pair { display:grid; grid-template-columns:280px minmax(0,1fr); grid-template-rows:minmax(0,1fr); gap:10px; width:100%; align-items:stretch; margin-top:10px; }
  .vault-pair .vault-files { grid-column:1; grid-row:1; min-width:0; }
  .vault-pair .vault-editor { grid-column:2; grid-row:1; min-width:0; }
  /* The Vault's own frame (0.1.9e Build 11, issue #134): the
     view is height-bound the way Docs is (#119) — the head
     rows (title, intro, search, recent) are fixed flex heads
     and the pair flexes into ALL the height that remains, so
     the workspace reaches the bottom of the window instead
     of ending where its content did. The files panel scrolls
     its own tree (the aside/.docs-wiki-nav pattern); the
     editor is a flex column — the textarea, or the preview,
     takes the height, and the links strip rides the column's
     foot. */
  #view-root.vault-view { display:flex; flex-direction:column; overflow:hidden; }
  #view-root.vault-view > * { flex:0 0 auto; }
  #view-root.vault-view > .vault-pair { flex:1 1 auto; min-height:0; }
  #view-root.vault-view .vault-pair > .card { margin-bottom:0; }
  .vault-files { overflow-y:auto; overscroll-behavior:contain; scrollbar-gutter:stable; min-height:0; }
  .vault-editor { display:flex; flex-direction:column; min-height:0; }
  .vault-editor > * { flex:0 0 auto; }
  #vault-ed { flex:1 1 auto; min-height:0; resize:none; }
  .vault-editor .vault-preview { flex:1 1 auto; min-height:0; overflow-y:auto; overscroll-behavior:contain; }
  /* The preview card is the editor card drawn again (a .card
     inside the .card): it IS the note in the viewer state
     (#284) — the textarea replaces it whole in the editing
     state, never beside it, so each takes the column's
     whole free height (the flex rules above) and the strip
     rides the foot either way. The press that opens the
     editor is the text's own cursor. */
  .vault-preview[data-vault-pressedit] { cursor:text; }
  /* Vault focus mode (0.1.9e Build 11, issue #135): the
     editor alone in the window — the app rail, the status
     column, the presence bar and the Vault's own head rows
     yield, and the pair collapses to the editor. The state
     lives in vaultFocus and nowhere else; the class on #app
     is render()'s reflection of it — the discipline of the
     pane button's face (#130): one state, read from the one
     place it is rendered. */
  #app.vault-focus, #app.vault-focus.closed, #app.vault-focus.no-status, #app.vault-focus.no-status.closed { grid-template-columns:minmax(0,1fr); }
  #app.vault-focus nav, #app.vault-focus aside { display:none; }
  #app.vault-focus main { grid-column:1 / -1; }
  #app.vault-focus #presence-bar { display:none; }
  #app.vault-focus .vault-head { display:none; }
  #app.vault-focus .vault-pair { grid-template-columns:minmax(0,1fr); margin-top:0; }
  #app.vault-focus .vault-pair .vault-files { display:none; }
  #app.vault-focus .vault-pair .vault-editor { grid-column:1; }
  #vault-focus-btn { min-width:40px; min-height:40px; }
  /* The Vault's mobile furniture (v0.5.9e, #279): the New
     button and the reader's back button exist in the markup
     at every width and show only in the phone block below —
     the desktop desk never sees them. The create sheet rides
     the #270 overlay pattern (fixed, portaled to <body>):
     a centered panel on a wide screen (it can only be opened
     from the phone furniture, but the shape is defined
     whole), a bottom sheet on a phone. */
  .vault-new-open { display:none; }
  .vault-back { display:none; }
  #vault-shade { position:fixed; inset:0; z-index:120; background:rgba(0,0,0,.45); }
  #vault-sheet { position:fixed; z-index:121; left:50%; top:50%; transform:translate(-50%,-50%); width:min(520px,calc(100vw - 48px)); max-height:86vh; overflow-y:auto; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:16px 18px 18px; animation:notif-in .65s cubic-bezier(.22,.8,.3,1); }
  /* Studio focus mode (0.1.9e Build 13, issue #146): the
     Vault's #135 idiom in the same shape — the app rail, the
     status column and the presence bar yield, and inside the
     Studio its header rows, pane switch and docs tail yield
     with them, leaving the panes and (on a phone) the thumb
     action bar. The state lives in studioFocus and nowhere
     else; the class on #app is render()'s reflection of it. */
  #app.studio-focus, #app.studio-focus.closed, #app.studio-focus.no-status, #app.studio-focus.no-status.closed { grid-template-columns:minmax(0,1fr); }
  #app.studio-focus nav, #app.studio-focus aside { display:none; }
  #app.studio-focus main { grid-column:1 / -1; }
  #app.studio-focus #presence-bar { display:none; }
  #app.studio-focus .studio-head, #app.studio-focus .studio-switch, #app.studio-focus .studio-tail { display:none; }
  #app.studio-focus #st-forge, #app.studio-focus .studio-made { display:none; }
  #app.studio-focus .studio-root { height:100%; display:flex; flex-direction:column; min-height:0; }
  #app.studio-focus .studio-panes { flex:1 1 auto; min-height:0; overflow-y:auto; }
  .msg { margin-bottom:10px; } .msg .who { font-size:11px; color:var(--dim); letter-spacing:.1em; }
  /* Issue #116: chat faces — the gutter avatar rides a flex row;
     the presence header wears her bundled portrait. */
  .msg-row { display:flex; gap:8px; align-items:flex-start; }
  .msg-body { flex:1 1 auto; min-width:0; }
  .m-av { width:28px; height:28px; border-radius:50%; object-fit:cover; flex:0 0 auto; margin-top:1px; background:var(--panel2); }
  span.m-av { display:inline-flex; align-items:center; justify-content:center; font-size:10px; color:var(--dim); border:1px solid var(--line); }
  .presence-av { width:34px; height:34px; border-radius:50%; object-fit:cover; flex:0 0 auto; }
  .exec { background:var(--exec-bg,#000); border:1px solid var(--line); border-radius:4px; padding:8px; white-space:pre-wrap; color:#a7f3d0; }
  /* CLI Mode shell sessions (Build 3, #209): the streamed record,
     the session header, and the desktop full-screen pane. On a
     phone the pane never renders \u2014 the streamed record below
     is the honest degradation, and it says so. (Build 5, #219:
     the Build 3 session input bar is retired \u2014 the composer
     is the session's only input; status and Close live in the
     header and the pane head.) */
  .wire-pills-body { animation:view-in var(--motion-med) var(--motion-ease); }
  .shellrec { background:var(--exec-bg,#000); border:1px solid var(--line); border-radius:4px; padding:8px; }
  .shell-out { margin:0; white-space:pre-wrap; word-break:break-word; color:#a7f3d0; font:inherit; }
  .shell-in { color:var(--text); margin:2px 0; word-break:break-word; }
  .shell-badge { border:1px solid var(--line); border-radius:4px; padding:0 4px; font-size:10px; letter-spacing:.06em; }
  .shell-badge.navi { color:var(--cy); border-color:var(--cy); }
  .shell-badge.op { color:var(--dim); }
  .shell-note { color:var(--dim); font-size:12px; font-style:italic; margin:2px 0; }
  .shellhead { display:flex; gap:8px; align-items:center; flex-wrap:wrap; border:1px solid var(--line); border-radius:6px; padding:8px; margin:0 0 8px; background:var(--panel2); }
  .shellhead button { min-height:44px; }
  .shellhead .shell-close { margin-left:auto; }
  .shellpane { border:1px solid var(--cy); border-radius:6px; margin:10px 0 0; background:#000; }
  .shellpane-head { display:flex; gap:10px; align-items:center; flex-wrap:wrap; color:var(--cy); font-size:11px; letter-spacing:.08em; padding:6px 10px; border-bottom:1px solid var(--line); }
  .shellpane-head .shell-close { margin-left:auto; min-height:32px; }
  .shellpane pre { margin:0; padding:10px; color:#a7f3d0; font-size:12px; line-height:1.35; overflow:auto; white-space:pre; min-height:200px; }
  .shellpane:focus { outline:2px solid var(--cy); }
  .shell-mobile-note { display:none; }
  @media (max-width:800px) {
    .shellpane { display:none; }
    .shell-mobile-note { display:block; color:var(--dim); font-size:12px; margin-top:6px; }
  }
  body[data-theme="casefile"] .exec { color:#f5ecd7; }
  .execstat { font-size:11px; color:var(--dim); padding:2px 2px 0; }
  .foldcard { border:1px solid var(--line); border-radius:6px; background:var(--panel); padding:6px 10px; margin:4px 0; }
  .foldcard summary { cursor:pointer; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .acsearch { border:1px solid var(--cy); border-radius:4px; padding:6px 8px; font-size:12px; background:var(--panel); }
  /* CLI-first composer: the syntax colors are shared by the typed
     line's highlight layer and the thread's terminal blocks, so a
     command reads the same while typed and once echoed. */
  .tk-cmd { color:var(--syn-cmd); } .tk-bad { color:var(--syn-bad); } .tk-flag { color:var(--syn-flag); } .tk-str { color:var(--syn-str); } .tk-path { color:var(--syn-path); } .tk-kw { color:var(--syn-kw); } .tk-num { color:var(--syn-num); } .tk-op { color:var(--syn-op); }
  .hlayer { position:absolute; inset:1px; padding:8px; white-space:pre; overflow:hidden; pointer-events:none; color:var(--text); font-family:ui-monospace,Menlo,Consolas,monospace; font-size:14px; line-height:1.5; }
  .hlayer .ghost { color:var(--dim); opacity:.55; }
  input.rich { position:relative; background:transparent; color:transparent; caret-color:var(--text); font-family:ui-monospace,Menlo,Consolas,monospace; font-size:14px; line-height:1.5; }
  input.rich::placeholder { color:var(--dim); opacity:1; }
  .modehint { display:flex; align-items:center; gap:8px; flex-wrap:wrap; padding-top:6px; }
  #view-root.chat-view > .modehint { flex:0 0 auto; }
  .modehint button { font-family:ui-monospace,Menlo,Consolas,monospace; font-size:11px; border:1px solid var(--line); color:var(--dim); background:none; border-radius:999px; padding:2px 10px; cursor:pointer; }
  .modehint button.cli { border-color:#34d399; color:#34d399; }
  .modehint .modegroup { display:inline-flex; border:1px solid var(--line); border-radius:999px; overflow:hidden; }
  .modehint .modegroup button { border:none; border-radius:0; font-weight:700; }
  .modehint .modegroup button.on-cli { color:#34d399; background:rgba(52,211,153,.13); }
  .modehint .modegroup button.on-navi { color:var(--cy); background:var(--cybg, rgba(34,211,238,.12)); }
  .modehint button.ghost { font-family:inherit; font-size:12px; }
  .docs-blocked a { color:#f5b544; font-weight:700; }
  /* Wiki topics: each heading is a dropdown — its pages expand and
     collapse with a smooth grid-rows glide, never a snap. The
     chevron turns with the state. */
  .doc-topic-head { display:flex; align-items:center; justify-content:space-between; gap:8px; width:100%; background:none; border:none; padding:2px 4px; margin:0; cursor:pointer; color:var(--dim); font:inherit; font-size:10px; font-weight:700; letter-spacing:.22em; text-transform:uppercase; text-align:left; }
  .doc-topic-head:hover { color:var(--text); }
  .doc-chev { display:inline-block; transition:transform var(--motion-glide) ease; font-size:11px; letter-spacing:0; }
  .doc-topic-head[aria-expanded="false"] .doc-chev { transform:rotate(-90deg); }
  .doc-topic-pages { display:grid; grid-template-rows:1fr; transition:grid-template-rows var(--motion-glide) ease; }
  .doc-topic-pages.closed { grid-template-rows:0fr; }
  .doc-topic-inner { overflow:hidden; min-height:0; }
  @media (prefers-reduced-motion: reduce) { .doc-topic-pages, .doc-chev { transition:none; } }
  .modehint span { color:var(--dim); font-size:11px; }
  /* Composer cleanup (build .66): the composer's whole zone — the
     chips row, the bar itself, the mode-toggle row — is one dock,
     not a stack: a hairline top border, the lifted panel tone
     (--panel stands off the --bg message area in all twelve
     themes), and bottom padding that clears the phone safe area.
     It bleeds to the frame's edges (the negative margins cancel
     main's padding; the padding inside restores the inset). One
     geometry inside: 44px targets, a 6px radius on the bar and
     the square action buttons, full-round reserved for the chip
     pills and the CLI | Navi toggle. */
  .composer { border-top:1px solid var(--line); background:var(--panel); margin:10px -16px -16px; padding:10px 16px calc(10px + env(safe-area-inset-bottom, 0px)); }
  .composer .row { align-items:stretch; }
  /* Slim chips (build .66): the shelf's pills are 32px tall with
     smaller text and tight 8px gaps, scrolling sideways when the
     shelf outgrows the bar; the CHIPS label centers on the row.
     No chips installed, no row at all (the render omits it). */
  .chipsrow { display:flex; align-items:center; gap:8px; overflow-x:auto; padding-bottom:8px; scrollbar-width:thin; }
  /* ChipDeck (0.1.9e, #94): the binder — sleeves in a wrapping grid,
     each holding the ACTUAL picture (the cartridge bytes, rendered
     pixel-sharp) with its state and actions under it. */
  .deck-grid { display:flex; flex-wrap:wrap; gap:12px; margin-top:10px; }
  .sleeve { flex:0 1 188px; border:1px solid var(--line); border-radius:8px; background:var(--panel); padding:10px; min-width:0; }
  .sleeve-img { width:96px; height:112px; image-rendering:pixelated; border-radius:4px; display:block; }
  .sleeve-name { font-weight:700; margin-top:8px; overflow-wrap:anywhere; }
  .sleeve-state { color:var(--dim); font-size:12px; margin-top:2px; overflow-wrap:anywhere; }
  .deck-actions { display:flex; flex-wrap:wrap; gap:6px; margin-top:8px; }
  .deck-actions button, .deck-actions a { min-height:40px; display:inline-flex; align-items:center; }
  .deck-drop { margin-top:10px; border:1px dashed var(--line); border-radius:8px; padding:16px 12px; text-align:center; }
  /* Threat Intelligence Center (0.1.9e Build 15, issue #165):
     desktop-first three-region grid — triage, the record, the
     brief rail. The phone gets its own arrangement, not a
     squeeze: one column, the brief FIRST (order is the phone's
     whole point — the answer before the apparatus), then the
     queue, then the record. */
  .tic-grid { display:grid; grid-template-columns:minmax(260px, 340px) 1fr minmax(240px, 300px); gap:12px; align-items:start; margin-top:10px; }
  .tic-fresh { display:flex; flex-wrap:wrap; gap:6px; margin-top:8px; }
  .tic-chip { border:1px solid var(--line); border-radius:999px; padding:6px 10px; font-size:12px; min-height:32px; display:inline-flex; align-items:center; }
  .tic-qitem { width:100%; text-align:left; margin-top:6px; min-height:44px; }
  @media (max-width: 800px) {
    .tic-grid { display:flex; flex-direction:column; }
    .tic-rail { order:-1; width:100%; }
    .tic-queue, .tic-detail { width:100%; }
    .tic-qitem { min-height:48px; }
  }
  @media (max-width: 800px) {
    .sleeve { flex:1 1 42%; }
    .deck-actions button, .deck-actions a { min-height:44px; }
  }
  /* Chip Studio phone elements (0.1.9e Build 13, issue #146):
     rendered always, visible only at phone widths — above
     800px the Studio is pixel-for-pixel the desktop it was. */
  .studio-switch, .studio-actions, .studio-snips, .studio-ac, .studio-face-toggle, .studio-run-out { display:none; }
  @media (max-width: 800px) {
    /* One pane at a time under a full-width segmented switch;
       the face as a collapsible strip; snippet shelf and an
       autocomplete row as the primary input aids; Run / Forge /
       Save in a bottom action bar that RESERVES its height in
       the flow (the #133 line — nothing slides under fixed
       chrome; the bar is a flex row of the bounded root, never
       position:fixed). The frame reserves its true height:
       head, strip, switch and bar are flex:0 rows and the
       panes take exactly what remains, scrolling inside. */
    #view-root:has(.studio-root) { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; display:flex; flex-direction:column; }
    #view-root:has(.studio-root) > .tabbar { flex:0 0 auto; }
    .studio-root { flex:1 1 auto; min-height:0; display:flex; flex-direction:column; }
    #app.studio-focus .studio-root { height:auto; }
    .studio-head { flex:0 0 auto; }
    .studio-face { flex:0 0 auto; }
    .studio-face-toggle { display:flex; width:100%; min-height:44px; align-items:center; margin-top:8px; }
    .studio-face .studio-face-row { display:none; }
    .studio-face.open .studio-face-row { display:flex; }
    .studio-switch { display:flex; gap:6px; width:100%; box-sizing:border-box; flex:0 0 auto; margin-top:8px; }
    .studio-switch button { flex:1 1 0; min-height:44px; }
    .studio-switch button.act, .studio-switch button.ghost { min-height:44px; }
    .studio-panes { flex:1 1 auto; min-height:0; overflow-y:auto; margin-top:8px; }
    .studio-pane { display:none; }
    .studio-pane.act { display:block; }
    .studio-snips { display:flex; gap:6px; flex-wrap:wrap; margin-bottom:8px; }
    .studio-snips button { min-height:44px; }
    .studio-snips button.ghost { min-height:44px; }
    .studio-ac { display:flex; gap:6px; flex-wrap:wrap; flex:0 0 auto; margin-top:8px; }
    .studio-ac:empty { display:none; }
    .studio-ac button { min-height:44px; }
    .studio-ac button.ghost { min-height:44px; }
    .studio-actions { display:flex; gap:8px; flex:0 0 auto; margin-top:8px; }
    .studio-actions button { flex:1 1 0; min-height:48px; }
    .studio-actions button.act, .studio-actions button.ghost { min-height:48px; }
    .studio-run-out { display:block; }
    #st-forge { display:none; }
    .studio-root .ed-scroll { height:300px; }
  }
  /* Guardian (0.1.9e Build 14, issue #156): the system
     monitor's face — desktop-first cards and tables; the
     dedicated phone interface for this view lives in the
     small-screen block below, stated with Guardian classes. */
  .guardian-stats { display:flex; flex-wrap:wrap; gap:10px 26px; margin-top:8px; }
  .guardian-stat { min-width:160px; }
  .guardian-stat .guardian-k { display:block; color:var(--dim); font-size:11px; letter-spacing:.12em; text-transform:uppercase; margin-bottom:2px; }
  .guardian-chip { display:inline-block; padding:2px 10px; border-radius:999px; border:1px solid var(--line); font-size:12px; font-weight:700; color:var(--dim); }
  .guardian-chip-ok { color:var(--green); border-color:var(--green); }
  .guardian-chip-tight { color:var(--am); border-color:var(--am); }
  .guardian-chip-critical { color:var(--red); border-color:var(--red); }
  .guardian-table-wrap { overflow-x:auto; max-width:100%; }
  .guardian-table { width:100%; border-collapse:collapse; font-size:13px; }
  .guardian-table th { text-align:left; color:var(--dim); font-size:11px; letter-spacing:.1em; text-transform:uppercase; padding:6px 8px; border-bottom:1px solid var(--line); }
  .guardian-table td { padding:6px 8px; border-bottom:1px solid var(--line); vertical-align:top; }
  .guardian-table tbody tr:last-child td { border-bottom:none; }
  .guardian-row.guardian-foreign td { color:var(--dim); }
  .guardian-badge { display:inline-block; padding:1px 8px; border-radius:999px; border:1px solid var(--am); color:var(--am); font-size:11px; font-weight:700; white-space:nowrap; }
  .guardian-badge-ours { border-color:var(--cy); color:var(--cy); }
  .guardian-fields { display:flex; flex-wrap:wrap; gap:8px 20px; margin-top:6px; }
  .guardian-fields label { display:flex; flex-direction:column; gap:4px; font-size:12px; color:var(--dim); }
  .guardian-settings .guardian-check { display:flex; align-items:center; gap:8px; font-size:13px; margin-top:10px; cursor:pointer; }
  .guardian-settings .guardian-check input { width:18px; height:18px; flex:0 0 auto; }
  .guardian-expendable { width:100%; box-sizing:border-box; margin-top:4px; }
  .guardian-log { list-style:none; margin:0; padding:0; }
  .guardian-log li { padding:6px 0; border-bottom:1px solid var(--line); font-size:13px; }
  .guardian-log li:last-child { border-bottom:none; }
  .boot-msg { font-size:12px; margin-top:6px; }
  .boot-linger-cmd { font-size:12px; margin-top:6px; overflow-wrap:anywhere; }
  .chips-label { color:var(--dim); font-size:11px; letter-spacing:.14em; flex:0 0 auto; }
  .chipbtn.chippill { height:32px; min-height:32px; margin:0; padding:0 12px; font-size:12px; display:inline-flex; align-items:center; flex:0 0 auto; }
  .composer #chat-in { height:44px; border-radius:6px; }
  /* The composer bar takes the entire width available to it
     (Build 5, #219): the input's wrap is the row's one growing
     member and the input fills it completely — no intrinsic
     sizing caps it, in either mode, at any width. */
  .composer .row > .ac-wrap { flex:1 1 auto; min-width:0; }
  .composer #chat-in { width:100%; }
  .composer #chat-upload, .composer #chat-mic { width:44px; height:44px; min-height:44px; padding:0; display:inline-flex; align-items:center; justify-content:center; border-radius:6px; flex:0 0 auto; }
  #chat-call { display:inline-flex; align-items:center; justify-content:center; gap:6px; height:44px; min-height:44px; padding:0 14px; border-radius:6px; flex:0 0 auto; }
  /* New case from chat (v0.4.5e, #241): the composer's case
     door wears the Call button's shape — 44px floor, word
     beside the glyph — and its form is a dialog in the Wire
     reader's idiom: shade + centered panel on desktop, the
     full phone screen under the phone chrome. */
  #chat-newcase { display:inline-flex; align-items:center; justify-content:center; gap:6px; height:44px; min-height:44px; padding:0 14px; border-radius:6px; flex:0 0 auto; }
  #nc-shade { position:fixed; inset:0; z-index:118; background:rgba(0,0,0,.45); }
  #nc-dialog { position:fixed; z-index:119; left:50%; top:50%; transform:translate(-50%,-50%); width:min(560px,calc(100vw - 48px)); max-height:86vh; overflow-y:auto; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:16px 18px calc(14px + env(safe-area-inset-bottom, 0px)); }
  #nc-dialog .nc-check { display:flex; gap:8px; align-items:flex-start; padding:6px 8px; border:1px solid var(--line); border-radius:8px; margin-top:6px; }
  #nc-dialog .nc-check input { width:20px; height:20px; margin-top:1px; flex:0 0 auto; }
  /* Mode feels alive (build .66): CLI latched, the bar takes the
     terminal green (--green in every theme) for its edge; Navi
     latched keeps the line color it has always had, under her
     cyan toggle. The class rides the autocomplete wrap, toggled
     by the same paint pass that dresses the toggle buttons. */
  .ac-wrap.cli-mode #chat-in { border-color:var(--green); }
  /* The relocated hint (build .66): plain muted words beside the
     toggle — "| Press enter to send." — never a button, never a
     paragraph of its own again. */
  .modehint .press-hint { color:var(--dim); font-size:12px; }
  .modehint .modegroup button { padding:4px 12px; }
  /* Docs page picker (build .66): phone widths only — the topic
     list hides and this one full-width select, grouped by the
     same topics, chooses the page instead. Desktop never sees
     it; the media block below turns it on. */
  .docs-picker { display:none; }
  input[type=text], textarea, select { background:var(--panel2); border:1px solid var(--line); color:var(--text); border-radius:4px; padding:8px; width:100%; }
  /* Native <select> widgets in some webviews (the Tauri shell's
     WebKitGTK among them) paint a white bar and ignore the themed
     background, leaving the near-white themed text unreadable on it —
     the chip picker, template picker, and editor Outline all showed
     it. Selects drop the native widget, draw their own chevron, and
     their option lists follow the theme; placeholders everywhere use
     the full-strength dim color instead of a faded default. */
  select { appearance:none; -webkit-appearance:none; background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6'%3E%3Cpath d='M1 1l4 4 4-4' stroke='%239fb3c8' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E"); background-repeat:no-repeat; background-position:right 8px center; background-size:10px 6px; padding-right:26px; }
  select option { background:var(--panel2); color:var(--text); }
  ::placeholder { color:var(--dim); opacity:1; }
  /* No focus zoom (Alex's rule, every view): mobile browsers zoom
     in whenever a focused field's text is under 16px — tapping the
     chat bar lurched the whole page. The viewport pins
     maximum-scale=1 with scaling off, manipulation touch-action
     kills double-tap zoom, and on touch devices every editable
     control renders at 16px, the threshold below which the focus
     zoom triggers. The composer's highlight layer (.hlayer) steps
     up with its input so the pair stay aligned; the Studio code
     editor is excluded because it sizes itself (16px by default
     on touch) in step with its own highlight layer. */
  html { touch-action:manipulation; }
  @media (pointer:coarse) {
    input:not([type=checkbox]):not([type=radio]):not([type=range]):not([type=file]):not(#st-code), textarea:not(#st-code), select { font-size:16px !important; }
    .hlayer { font-size:16px !important; }
  }
  .row { display:flex; gap:8px; } .row > * { flex:0 1 auto; } .row .grow { flex:1 1 auto; }
  .row > button, .ed-bar button { white-space:nowrap; flex:0 0 auto; }
  /* Shared tab bar (Alex's rule, every tabbed view): the bar spans
     its container's full width, carries a small buffer at both
     ends and a small gap between tabs, and the tabs divide what
     remains equally — re-dividing on their own when a tab is
     added or removed. One style, everywhere tabs appear. */
  .tabbar { display:flex; gap:6px; width:100%; padding-left:6px; padding-right:6px; box-sizing:border-box; }
  /* NUR flip cards: a grid of fixed-size landscape cards — the
     front carries the install state and actions; the back repeats
     the basics and scrolls the README inside the card. The card
     never changes size with the README. Inside the fixed height,
     the front's text lines are flex:none with explicit line-
     heights: a full card shortens its description/release lines
     with an ellipsis, it never slices a line mid-glyph. The
     front's buttons sit in two columns: the secondary actions
     flow in the flexible left column (.flip-side), and the right
     column (.flip-main) — a fixed-width stack anchored to the
     card's right edge — carries Install/Update on top with the
     README button directly beneath it. The height is 212px:
     with the buttons beside the info instead of stacked under
     it, the front's content (header, two-line description,
     stats and release lines, the two-column action row) needs
     only ~207px, so the card sits at a shorter, more landscape,
     cartridge-like proportion with the slack left in the auto
     margin above the actions — never in a squeezed line. */
  .nur-grid { display:grid; grid-template-columns:repeat(auto-fill, minmax(290px, 1fr)); gap:10px; }
  .flip-card { perspective:1200px; height:212px; }
  .flip-inner { position:relative; width:100%; height:100%; transform-style:preserve-3d; transition:transform var(--motion-slow) ease; }
  .flip-card.flipped .flip-inner { transform:rotateY(180deg); }
  .flip-face { position:absolute; inset:0; backface-visibility:hidden; -webkit-backface-visibility:hidden; overflow:hidden; display:flex; flex-direction:column; background:var(--panel); border:1px solid var(--line); border-radius:8px; padding:8px 12px; box-sizing:border-box; }
  .flip-back { transform:rotateY(180deg); }
  .flip-scroll { flex:1; min-height:0; overflow-y:auto; margin-top:8px; padding-right:4px; }
  .flip-foot { margin-top:auto; padding-top:8px; display:flex; flex-wrap:wrap; gap:6px; align-items:center; flex:none; }
  .flip-actions { margin-top:auto; padding-top:5px; flex:none; display:flex; gap:8px; align-items:flex-start; }
  .flip-side { flex:1 1 auto; min-width:0; display:flex; flex-wrap:wrap; gap:6px; align-items:flex-start; }
  .flip-main { display:flex; flex-direction:column; align-items:stretch; gap:4px; flex:0 0 9rem; }
  .flip-main .act, .flip-main .ghost { width:100%; text-align:center; box-sizing:border-box; }
  .flip-main .act { padding-top:5px; padding-bottom:5px; }
  .flip-main .ghost { padding-top:4px; padding-bottom:4px; }
  /* Local Chips cards: the same two-column split — secondary
     actions in the flexible left column, the primary action
     (Run) on top of a fixed-width right column with the chip
     file's Download .png button directly beneath it. The right
     column's buttons carry the same slimmed paddings as the
     catalog cards' .flip-main, so these cards sit shorter too. */
  .lchip-actions { display:flex; gap:8px; align-items:flex-start; margin-top:6px; }
  .lchip-side { flex:1 1 auto; min-width:0; display:flex; flex-wrap:wrap; gap:6px; align-items:center; }
  .lchip-main { display:flex; flex-direction:column; align-items:stretch; gap:4px; flex:0 0 9rem; }
  .lchip-main .act, .lchip-main .ghost { width:100%; text-align:center; box-sizing:border-box; }
  .lchip-main .act { padding-top:6px; padding-bottom:6px; }
  .lchip-main .ghost { padding-top:5px; padding-bottom:5px; }
  @media (prefers-reduced-motion: reduce) { .flip-inner { transition:none; } }
  .tabbar > * { flex:1 1 0; min-width:0; }
  button.act { background:var(--cy); color:#000; border:0; font-weight:700; padding:8px 14px; border-radius:4px; cursor:pointer; }
  button.ghost { background:none; color:var(--text); border:1px solid var(--line); padding:6px 10px; border-radius:4px; cursor:pointer; }
  a.act, a.ghost { display:inline-block; text-decoration:none; font:inherit; box-sizing:border-box; }
  a.act { background:var(--cy); color:#000; border:0; font-weight:700; padding:8px 14px; border-radius:4px; cursor:pointer; }
  a.ghost { background:none; color:var(--text); border:1px solid var(--line); padding:6px 10px; border-radius:4px; cursor:pointer; }
  .chipbtn { border:1px solid var(--cy); color:var(--cy); background:none; border-radius:999px; padding:3px 10px; cursor:pointer; margin:0 4px 4px 0; }
  .srcs { display:flex; flex-wrap:wrap; gap:4px; margin-top:5px; }
  /* Build 8 (#88): the one pill row under a reply never wraps —
     Mesh pill first, then the source pills, all on one line.
     Inside the row the pills may shrink: the Mesh pill keeps its
     full text (flex none), and a source pill ellipsizes before
     it could push the row — and with it the page — sideways at
     phone widths. */
  .srcs.pillrow { flex-wrap:nowrap; min-width:0; }
  .srcs.pillrow .meshpill { flex:0 0 auto; }
  .srcs.pillrow .srcpill { min-width:0; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  .srcpill { font-family:ui-monospace,Menlo,Consolas,monospace; font-size:11px; color:var(--dim); border:1px solid var(--line); border-radius:999px; padding:2px 9px; }
  .srcpill .srcsep { color:var(--cy); }
  /* The Mesh attribution pill (Build 3): a borrowed reply's
     "who wrote this" as one compact pill in Mesh Cyan — #22D3EE,
     the signature theme's --cy — modeled on .srcpill's geometry.
     Text and border ride the theme's cyan slot (every theme
     tunes it readable on its own surfaces; signature, glass,
     nocturne and abyss hold #22D3EE itself) over a translucent
     cyan ground — the fixed rgba is Mesh Cyan at 10%, and
     color-mix re-derives the ground from the theme's cyan where
     the engine supports it. */
  .meshpill { display:inline-flex; align-items:center; gap:5px; font-family:ui-monospace,Menlo,Consolas,monospace; font-size:11px; color:var(--cy); border:1px solid rgba(34,211,238,.45); border-color:color-mix(in srgb, var(--cy) 55%, transparent); border-radius:999px; padding:2px 9px; background:rgba(34,211,238,.10); background:color-mix(in srgb, var(--cy) 12%, transparent); }
  /* Build 2 (#203): the Fedinet route pill — the same pill,
     gone federation green, reading the recorded route:
     Fedinet > carrier (when one carried it) > Answered by.
     On a phone the route truncates from the middle out: the
     carrier segment elides first; the fedinet and the
     answerer always survive. */
  .meshpill.fedpill { color:var(--green); border-color:color-mix(in srgb, var(--green) 55%, transparent); background:color-mix(in srgb, var(--green) 12%, transparent); }
  /* The route pill (v0.4.1e, #228): the composer's one route
     control. Its face IS the bridge's stored route for the
     chat — Local neutral, Mesh blue, Federation green in the
     fedinet's name, and a mode whose access was lost reads
     Not Available in amber rather than claiming a road that
     is not there. */
  .modehint .routepill { font-weight:700; min-height:32px; padding:4px 12px; }
  .modehint .routepill-mesh { border-color:color-mix(in srgb, var(--blue) 60%, transparent); color:var(--blue); background:color-mix(in srgb, var(--blue) 10%, transparent); }
  .modehint .routepill-fed { border-color:color-mix(in srgb, var(--green) 60%, transparent); color:var(--green); background:color-mix(in srgb, var(--green) 10%, transparent); }
  .modehint .routepill-na { border-color:color-mix(in srgb, var(--am) 60%, transparent); color:var(--am); }
  .modehint .routepill-auto { border-style:dashed; }
  .modehint .route-msg { font-size:11px; color:var(--am); }
  @media (max-width:800px) { .modehint .routepill { min-height:44px; } }
  /* The Wire's own tab bar (v0.4.1e, #230): the four families
     as pages inside The Wire, one tap apart, ≥44px targets. */
  .wiretabs { display:flex; gap:8px; flex-wrap:wrap; margin-bottom:12px; }
  .wiretabs button { min-height:44px; padding:8px 16px; font-weight:700; }
  .wiretabs button .wt-count { font-family:ui-monospace,Menlo,Consolas,monospace; font-size:11px; font-weight:400; color:var(--dim); margin-left:6px; }
  .wiretabs button.act .wt-count { color:inherit; }
  /* Wire tab search + the pull picker (v0.5.10e, #283): the
     search input rides the tab's own width at every size —
     100% of its row, never wider, so the page itself never
     scrolls sideways on a phone. */
  .wiresearch input[type='search'] { background:var(--panel2); border:1px solid var(--line); border-radius:8px; color:var(--text); padding:8px 12px; font-size:14px; box-sizing:border-box; min-height:44px; }
  mark.wire-hit { background:rgba(245,181,68,.35); color:inherit; border-radius:3px; padding:0 1px; }
  #wire-pull-shade { position:fixed; inset:0; z-index:118; background:rgba(0,0,0,.45); }
  #wire-pull-picker { position:fixed; z-index:119; left:50%; top:50%; transform:translate(-50%,-50%); width:min(560px,calc(100vw - 32px)); max-height:86vh; overflow-y:auto; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:18px 20px; box-sizing:border-box; }
  .shelf-card { border-color:var(--am); }
  .meshpill.fedpill .fedseg-elide { display:none; }
  @media (max-width:800px) {
    .meshpill.fedpill .fedseg-mid { display:none; }
    .meshpill.fedpill .fedseg-elide { display:inline; }
  }
  pre.out { background:#000; border:1px solid var(--line); border-radius:4px; padding:8px; white-space:pre-wrap; overflow:auto; }
  .ac-wrap { position:relative; }
  .ac { position:absolute; left:0; right:0; bottom:calc(100% + 4px); background:var(--panel); border:1px solid var(--cy); border-radius:4px; overflow:hidden; z-index:20; box-shadow:0 8px 24px rgba(0,0,0,.5); }
  .ac-it { display:flex; gap:8px; align-items:baseline; padding:6px 8px; cursor:pointer; }
  .ac-it.on { background:rgba(34,211,238,.12); }
  .ac-it b { font-family:ui-monospace,Menlo,Consolas,monospace; }
  .ac-it span { color:var(--dim); font-size:12px; flex:1; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .ac-it em { color:var(--dim); font-size:9px; font-style:normal; letter-spacing:.12em; text-transform:uppercase; }
  .switch { width:44px; height:22px; border-radius:999px; border:1px solid var(--line); background:var(--panel); position:relative; cursor:pointer; padding:0; }
  .switch .knob { position:absolute; top:2px; left:2px; width:16px; height:16px; border-radius:50%; background:var(--dim); }
  .switch[aria-checked="true"] { border-color:var(--red); background:rgba(248,113,113,.15); }
  .switch[aria-checked="true"] .knob { left:24px; background:var(--red); }
  /* ---- polish: motion, hover highlights, press feedback ---- */
  #app { transition:grid-template-columns var(--motion-view) ease; }
  nav button, button.act, button.ghost, a.act, a.ghost, .chipbtn, .card, .switch .knob { transition:background var(--motion-fast) ease, color var(--motion-fast) ease, border-color var(--motion-fast) ease, filter var(--motion-fast) ease, transform var(--motion-press) ease, left var(--motion-fast) ease; }
  nav button:hover { background:rgba(34,211,238,.12); color:var(--text); }
  nav button.on:hover { background:var(--cy); color:#000; }
  button.ghost:not(#chat-status-toggle):hover, a.ghost:hover, .chipbtn:hover { border-color:var(--cy); color:var(--cy); }
  /* The pane button's hover is a background tint only (Build 9,
     #130): the state accent above is the one thing its face may
     ever say, and :hover must not be able to say it — on a phone
     the tap that opens the dock lands on this button, and a
     hover accent left behind would paint it "open" over a
     closed dock after any dismiss. Its color and border-color
     also leave the transition list (its own rule above), so
     the face changes with the state, never after it. */
  #chat-status-toggle:hover { background:rgba(34,211,238,.12); }
  button.act:hover, a.act:hover { filter:brightness(1.12); }
  button.act:active, button.ghost:active, a.act:active, a.ghost:active, .chipbtn:active, nav button:active { transform:scale(.97); }
  .card:hover { border-color:#2e4356; }
  @keyframes view-in { from { opacity:0; transform:translateY(6px); } to { opacity:1; transform:none; } }
  #view-root.view-in { animation:view-in var(--motion-view) ease both; }
  /* Presence pulse (the typing indicator that used to share
     this note was removed in v0.4.1e, #228 — the top banner
     carries no typing state anymore). */
  @keyframes navi-pulse { 0%,100% { opacity:1; } 50% { opacity:.35; } }
  .pdot { animation:navi-pulse 2.2s ease-in-out infinite; }
  /* Chip Studio code editor: highlighted pre behind a transparent
     textarea, sticky line-number gutter (same construction as the
     space's editor). */
  .ed-wrap { position:relative; border:1px solid var(--line); border-radius:6px; background:#0b0f14; overflow:hidden; }
  /* Build 13 (#146): the editor's old fixed-position fullscreen
     (.ed-wrap.fs) is RETIRED — focus mode (#app.studio-focus,
     beside the Vault's rules) is the one expand idiom app-wide. */
  .ed-guides, .ed-squigs, .ed-extras { position:absolute; top:0; left:0; right:0; bottom:0; pointer-events:none; overflow:hidden; }
  .ed-guides i { position:absolute; top:0; bottom:0; width:1px; background:var(--line); opacity:.45; }
  .ed-squigs i { position:absolute; height:3px; background:repeating-linear-gradient(45deg,#f87171 0 2px,transparent 2px 4px); }
  .ed-extras i { position:absolute; width:2px; background:#f472b6; }
  .ed-mini { flex:0 0 64px; border-left:1px solid var(--line); overflow:hidden; cursor:pointer; background:#0d1218; position:sticky; right:0; z-index:3; box-shadow:-6px 0 10px rgba(0,0,0,.35); }
  .ed-mini pre { margin:0; padding:4px; font:3px/1.4 ui-monospace,Menlo,Consolas,monospace; color:var(--dim); white-space:pre; }
  .ed-pal { position:absolute; top:38px; left:50%; transform:translateX(-50%); z-index:5; width:min(420px,90%); background:var(--panel); border:1px solid var(--line); border-radius:6px; padding:8px; box-shadow:0 12px 40px rgba(0,0,0,.5); }
  .ed-gutter button { background:none; border:0; color:var(--cy); cursor:pointer; font:inherit; padding:0 2px; }
  .ed-bar { display:flex; flex-wrap:wrap; gap:6px; align-items:center; border-bottom:1px solid var(--line); padding:6px 8px; }
  .ed-scroll { overflow:auto; height:380px; }
  .ed-inner { display:flex; min-height:100%; }
  .ed-gutter { position:sticky; left:0; z-index:2; background:#0d1218; color:var(--dim); text-align:right; padding:10px 8px 10px 12px; font:12px/1.55 ui-monospace,Menlo,Consolas,monospace; user-select:none; min-width:40px; flex:0 0 auto; }
  .ed-code { position:relative; flex:1 1 auto; min-width:max-content; }
  .ed-code pre, .ed-code textarea { margin:0; padding:10px 12px; font:12.5px/1.55 ui-monospace,Menlo,Consolas,monospace; white-space:pre; tab-size:2; }
  .ed-code pre { color:var(--text); }
  .ed-code textarea { position:absolute; inset:0; height:100%; width:100%; background:transparent; color:transparent; caret-color:var(--cy); border:0; border-radius:0; resize:none; overflow:hidden; outline:none; }
  .mded { position:relative; height:280px; border:1px solid var(--line); border-radius:6px; overflow:hidden; background:var(--panel2); }
  .mded pre, .mded textarea { margin:0; padding:10px 12px; font:12.5px/1.55 ui-monospace,Menlo,Consolas,monospace; white-space:pre-wrap; word-break:break-word; tab-size:2; }
  .mded pre { color:var(--text); position:absolute; inset:0; overflow:hidden; pointer-events:none; }
  .mded textarea { position:absolute; inset:0; height:100%; width:100%; background:transparent; color:transparent; caret-color:var(--cy); border:0; border-radius:0; resize:none; overflow:auto; outline:none; }
  .tok-com { color:var(--dim); font-style:italic; } .tok-str { color:var(--green); } .tok-kw { color:var(--cy); font-weight:600; } .tok-num { color:var(--am); } .tok-navi { color:#f472b6; font-weight:700; } .tok-call { font-weight:600; } .tok-dec { color:#a78bfa; }
  .api-m { border:1px solid var(--line); border-radius:6px; padding:8px 10px; margin-bottom:8px; }
  .wf-step { border:1px solid var(--line); border-radius:6px; padding:8px 10px; margin-bottom:8px; background:var(--panel2); }
  .doc-sec h4 { color:var(--text); letter-spacing:.14em; font-size:12px; text-transform:uppercase; margin:18px 0 6px; }
  .doc-sec p, .doc-sec li { color:var(--dim); line-height:1.6; }
  .doc-sec b, .doc-sec .mono { color:var(--text); }
  /* Chat markdown: Navi's replies render as real markdown — fenced
     code becomes a code block, inline code a chip, lists and
     headings structure the prose instead of dumping raw backticks. */
  .msg { animation:view-in var(--motion-view) ease both; }
  .md p { margin:0 0 8px; } .md p:last-child { margin-bottom:0; }
  .md .ic { background:rgba(34,211,238,.13); border:1px solid var(--line); border-radius:4px; padding:0 5px; font-family:ui-monospace,Menlo,Consolas,monospace; font-size:.92em; color:var(--cy); }
  .md-code { background:var(--exec-bg); border:1px solid var(--line); border-radius:4px; padding:8px; white-space:pre-wrap; color:var(--text); margin:6px 0; overflow:auto; font-size:12.5px; }
  .md-twrap { overflow-x:auto; max-width:100%; margin:6px 0; }
  .md-t { border-collapse:collapse; font-size:13px; }
  .md-t th, .md-t td { border:1px solid var(--line); padding:4px 8px; text-align:left; vertical-align:top; }
  .md-t th { color:var(--text); background:var(--panel2); }
  .md-ul { margin:4px 0 8px; padding-left:20px; }
  .md-h { color:var(--text); margin:10px 0 4px; font-size:15px; letter-spacing:.02em; text-transform:none; }
  .md-img { max-width:100%; height:auto; display:block; border-radius:8px; margin:6px 0; }
  /* The Dossier (v0.5.6e, #260): tabs across the top — the
     strip scrolls sideways on a phone instead of wrapping
     or shrinking; every target keeps the 44px floor. */
  .dz-tabs { display:flex; gap:6px; overflow-x:auto; padding:2px 0 6px; scrollbar-width:thin; }
  .dz-tabs > * { flex:0 0 auto; min-height:44px; }
  .dz-run input, .dz-run button, .dz-run select { min-height:44px; }
  @media (max-width:800px) {
    .dz-tabs { -webkit-overflow-scrolling:touch; }
    .dz-run .row { flex-wrap:wrap; }
    /* v0.5.12e (#285): the open case's action row wraps at
       phone widths — every action stays whole and tappable,
       and the row never overflows its card. */
    .cf-actions { flex-wrap:wrap; }
    .cf-actions > button { flex:1 1 auto; }
  }
  /* FediChat as a chat surface (v0.5.5e, #270): the room is a
     height-bound frame, not a stack of cards — the frame class
     lands on #view-root like chat-view / docs-view / vault-view,
     the lines scroll inside their card, and the composer is
     docked at the frame's foot. The members rail and the
     header/settings cards left the page flow: the users live
     in a tray that slides over the room from the left, and the
     room's facts and settings live in a floating window —
     both portaled to <body> while open, in the wire reader's
     pattern (#196), so their fixed geometry anchors to the
     viewport on every door. */
  #view-root.fedichat-view { display:flex; flex-direction:column; overflow:hidden; }
  #view-root.fedichat-view > * { flex:0 0 auto; }
  #view-root.fedichat-view > .fc-room { flex:1 1 auto; min-height:0; }
  .fc-room { display:flex; flex-direction:column; min-height:0; width:100%; }
  .fc-topbar { display:flex; align-items:center; gap:8px; flex:0 0 auto; margin-bottom:8px; }
  .fc-topbar .fc-room-name { font-size:16px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  #fc-log { flex:1 1 auto; min-height:0; overflow-y:auto; overscroll-behavior:contain; margin-bottom:0; }
  .fc-line { padding:3px 0; border-bottom:1px solid rgba(127,127,127,.08); }
  .fc-complete { margin-top:8px; flex:0 0 auto; }
  .fc-complete:empty { display:none; }
  .fc-composer { flex:0 0 auto; margin-top:8px; }
  .fc-hint { flex:0 0 auto; }
  .fc-settings-phone { display:none; }
  #fc-shade { position:fixed; inset:0; z-index:120; background:rgba(0,0,0,.45); }
  #fc-tray { position:fixed; z-index:121; top:0; left:0; bottom:0; width:min(320px,86vw); overflow-y:auto; background:var(--panel2); border-right:1px solid var(--line); box-shadow:14px 0 40px rgba(0,0,0,.35); padding:14px 16px calc(14px + env(safe-area-inset-bottom)); animation:fc-tray-in var(--motion-med) var(--motion-ease); }
  @keyframes fc-tray-in { from { transform:translateX(-24px); opacity:0; } to { transform:none; opacity:1; } }
  /* FediChat Build 2 (v0.5.7e): quotes, media, leads, presence, search */
  .fc-line.hl { background:rgba(163,190,140,.16); border-radius:6px; }
  .fc-quote { border-left:3px solid var(--cy); padding:2px 8px; margin:2px 0 4px; color:var(--dim); font-size:12.5px; }
  .fc-quote-bad { border-left-color:#f87171; color:#fca5a5; }
  .fc-media img { max-width:min(420px,100%); border-radius:8px; display:block; margin-top:4px; }
  .fc-lead { border:1px solid var(--line); border-left:3px solid #b48ead; border-radius:8px; padding:6px 10px; margin:2px 0; }
  .fc-presence { color:var(--dim); font-size:12px; font-style:italic; min-height:16px; }
  .fc-searchrow { display:flex; gap:6px; margin-top:6px; }
  .fc-searchrow input { min-height:44px; }
  .fc-mini { font-size:11px; min-height:28px; padding:2px 8px; }
  .fc-linemeta { margin-top:2px; }
  #fc-win { position:fixed; z-index:121; left:50%; top:50%; transform:translate(-50%,-50%); width:min(560px,calc(100vw - 48px)); max-height:86vh; overflow-y:auto; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:16px 18px 18px; animation:notif-in .65s cubic-bezier(.22,.8,.3,1); }
  .fc-ov-head { display:flex; align-items:center; gap:8px; margin-bottom:8px; }
  .fc-ov-title { font-weight:700; font-size:15px; flex:1; min-width:0; }
  .fc-ov-x { min-width:44px; min-height:44px; margin:-6px -10px 0 0; background:none; border:0; color:var(--dim); font-size:20px; cursor:pointer; }
  @media (max-width:800px) {
    #fc-win { left:12px; right:12px; top:auto; bottom:12px; transform:none; width:auto; max-height:76vh; }
    #fc-settings { display:none; }
    .fc-settings-phone { display:block; width:100%; margin-top:8px; min-height:44px; flex:0 0 auto; }
    .fc-topbar button { min-height:44px; }
  }
  .md-q { border-left:3px solid var(--cy); margin:6px 0; padding:2px 10px; color:var(--dim); }
  .md a { color:var(--cy); }
  /* Voice: the call panel's live wave (bars rise while she listens
     or speaks) and the big round call button. */
  @keyframes navi-wave { 0%,100% { transform:scaleY(.35); } 50% { transform:scaleY(1); } }
  .wave { display:inline-flex; gap:3px; align-items:center; height:22px; }
  .wave i { width:4px; height:22px; border-radius:2px; background:var(--green); transform-origin:center; animation:navi-wave 1s ease-in-out infinite; }
  .wave i:nth-child(2) { animation-delay:.12s; } .wave i:nth-child(3) { animation-delay:.24s; } .wave i:nth-child(4) { animation-delay:.36s; } .wave i:nth-child(5) { animation-delay:.48s; }
  .wave.speaking i { background:var(--cy); }
  .wave.idle i { animation:none; transform:scaleY(.35); background:var(--dim); }
  .callbtn { width:76px; height:76px; border-radius:50%; border:2px solid var(--green); background:rgba(52,211,153,.12); color:var(--green); font-weight:700; cursor:pointer; font:inherit; }
  .callbtn.end { border-color:var(--red); background:rgba(248,113,113,.12); color:var(--red); }
  .callbtn:disabled { opacity:.45; cursor:default; }
  .ptt { border:1px solid var(--cy); color:var(--cy); background:none; border-radius:999px; padding:8px 18px; cursor:pointer; font:inherit; font-weight:700; }
  .ptt.rec { background:var(--red); border-color:var(--red); color:#000; }
  @media (prefers-reduced-motion: reduce) { .wave i { animation:none; } }
  @media (prefers-reduced-motion: reduce) { .tdots i, .pdot { animation:none; } #view-root.view-in { animation:none; } #app { transition:none; } .glitching { animation:none; } }
  /* Small screens — everything in this block is breakpoint-scoped;
     the desktop frame above it is untouched. Build .61 removed the
     fixed 100dvh grid (the cause of the old STATUS-over-chips
     overlap) for a normal-flow column. Build .64 (the mobile chrome
     round, #46's remaining scope) finishes the phone frame Alex
     directed: the nav is a slim PERSISTENT top bar — pinned by
     position:sticky, always on screen — carrying the center line
     "Apex | powered by Navi" between two toggles: the hamburger
     (#nav-toggle, the existing collapse state) on the left and the
     dock toggle (#dock-toggle) on the right. The menu overlay
     slides in from the left and the status dock from the right,
     each from its own toggle's side. The
     destinations live in #nav-menu as a full-phone-width overlay
     under the bar (on desktop that container is display:contents
     and nothing changes); the status column leaves the page flow
     entirely and opens from the dock toggle as a full-phone-width
     overlay in its locked order. Choosing a destination, following
     a link out of the dock, or tapping back into the content closes
     whichever overlay is open — the resting state on any page is
     content only, under the bar. Chat keeps its own frame — thread
     scrolling, composer pinned — sized to the viewport under the
     bar. Touch targets are finger-sized, and wide children wrap or
     scroll inside themselves, so nothing scrolls sideways at
     360–430px. Build .65 (Alex's Build 6 field test on his
     iPhone): the hamburger wraps only its icon — flex:0 0 auto,
     a square ≥44px target exactly like the dock toggle's, never
     flex-growing (the base nav-button width:100% is overridden
     here) — keeps the ☰ glyph whether the menu is open or
     closed (the ✕ swap is gone), and while the menu is open it
     takes the dock toggle's own open-state highlight (accent
     border, accent glyph), so the two bar buttons read as a
     pair. With the button compact, the one-line title beside it
     shows in full at 360–430px. */
  @media (max-width:800px) {
    html, body { height:auto; overflow:auto; }
    #app, #app.closed { display:flex; flex-direction:column; height:auto; min-height:calc(100dvh / var(--uiscale, 1)); overflow:visible; padding-top:60px; }
    nav { position:fixed; left:0; right:0; top:0; z-index:90; flex-direction:row; flex-wrap:nowrap; align-items:center; gap:6px; height:60px; overflow:visible; padding:6px 10px; border-right:0; border-bottom:1px solid var(--line); background:var(--panel2); }
    nav .logo, nav .tagline { display:none; }
    .bar-title { display:block; flex:1 1 auto; min-width:0; text-align:left; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; color:var(--text); font-size:13px; letter-spacing:.04em; }
    #nav-toggle { order:-1; flex:0 0 auto; width:auto; margin:0; min-width:44px; min-height:44px; padding:8px 10px; font-size:16px; justify-content:center; }
    #app:not(.closed) #nav-toggle { color:var(--cy); border-color:var(--cy); }
    #dock-toggle { display:flex; align-items:center; justify-content:center; width:auto; flex:0 0 auto; margin-left:auto; min-width:44px; min-height:44px; padding:8px 10px; color:var(--dim); border:1px solid var(--line); background:none; border-radius:4px; cursor:pointer; }
    #app.dock-open #dock-toggle { color:var(--cy); border-color:var(--cy); }
    nav button { min-height:44px; }
    #nav-menu { position:fixed; top:60px; left:0; right:0; bottom:0; z-index:85; display:flex; flex-direction:column; gap:4px; padding:10px; overflow-y:auto; background:var(--panel2); transform:translateX(-104%); visibility:hidden; transition:transform var(--motion-med) ease, visibility 0s linear var(--motion-med); }
    #app:not(.closed) #nav-menu { transform:translateX(0); visibility:visible; transition:transform var(--motion-med) ease; }
    #nav-menu button[id^="nav-"] { flex:0 0 auto; width:100%; min-height:48px; justify-content:flex-start; }
    /* Grouped menu on phones (Build 4): groups expand inline in
       the full-width overlay — no flyouts — children and headers
       alike meet the 48px row the other destinations get, and the
       Utilities children open upward within the same panel,
       never off-screen (the panel itself scrolls). */
    #nav-menu .nav-children-inner > button { min-height:48px; }
    #nav-menu .nav-group-sec { gap:4px; }
    #nav-menu .nav-group-head { min-height:48px; }
    #nav-menu .nav-util { margin-top:auto; }
    #nav-menu .nav-break { margin:6px 2px; }
    #app.closed nav button .lbl { display:inline; }
    #app.closed .nav-tip { display:none; }
    .nav-tip { display:none; }
    #nav-version { width:100%; }
    #app.closed #nav-version { text-align:center; padding:9px 8px 0; }
    #app.closed #nav-version .nv-pre { display:inline; }
    main { height:auto; min-height:0; overflow:visible; padding:12px; }
    /* The presence bar's button keeps its 44px sizing (the
       base rule above) on phones too. */
    #view-root.chat-view { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; overflow:hidden; }
    #view-root.docs-view { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; overflow:hidden; }
    #view-root.fedichat-view { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; }
    /* Chat's frame, height-bound (Build 10, issue #131): the
       height above reckons the chat view as the viewport minus
       the 60px top bar alone — but the view does not start
       under the bar; it starts under the presence bar too, and
       the frame carries main's own padding above and below.
       Those heights were never in the sum, so the frame stood
       taller than the screen by exactly them (78px at the
       field's geometry: 12 + 44 + 10 + 12), the composer's
       foot sat that far BELOW the screen's edge, and the page
       scroll — the phone model's own — was what carried it in
       and out of view. While Chat is up (render() marks the
       frame .in-chat), the frame takes the viewport's height
       exactly and the centre column and the view flex inside
       it: main is height-bound with the presence bar as its
       fixed head, the chat view flexes in what remains, and
       #msgs — already flex:1 with its own overflow — is the
       only scroll region. The composer's bottom is then the
       frame's rendered bottom by construction, whatever rides
       above it (the bar, a warning card), and the page has
       nothing left to scroll. Other views keep the page-flow
       model above; the composer's width is untouched. */
    #app.in-chat { height:calc(100dvh / var(--uiscale, 1)); }
    #app.in-chat main { flex:1 1 0; min-height:0; overflow:hidden; }
    #app.in-chat #view-root.chat-view { flex:1 1 auto; height:auto; min-height:0; }
    /* The frame's reach at the phone padding (issue #133,
       the base rule above restated): main pads 12px here. */
    #view-root.chat-view { margin-bottom:-12px; }
    /* The Vault's frame at phone widths takes the same height
       the Docs and Chat frames take in this model (issue
       #134): the viewport under the 60px top bar, so the view
       ends at the viewport's foot instead of standing one
       padding taller than the visible screen on the page's
       scroll. (Its base height is the flex share on desktop;
       this restates it for the page-flow phone model.) */
    #view-root.vault-view { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; }
    /* Her presence on a phone (0.1.9e Build 11, issue #140):
       the bar retires — the status pane's own head row has
       always carried the same line from the same painter, so
       the phone showed her twice. The pane row becomes her
       phone home: portrait and name beside the line, shown
       only in this block. The bar's pane button retires with
       the bar — the top bar's dock toggle is the phone's one
       door (the desktop keeps both of its doors, and #130's
       accent keying is untouched). The reclaimed height goes
       back to the content by the frame's own flex geometry:
       the in-chat bound never reckoned the bar as a term, so
       no sum anywhere changes. The bar element stays in the
       DOM and its painter keeps painting it — invisible, and
       the pane's line updates from the same source either
       way. */
    #presence-bar { display:none; }
    aside { position:fixed; top:60px; left:0; right:0; bottom:0; z-index:85; border-left:0; border-top:1px solid var(--line); background:var(--panel2); height:auto; max-height:none; min-height:0; overflow-y:auto; overscroll-behavior:contain; padding:12px; transform:translateX(104%); visibility:hidden; transition:transform var(--motion-med) ease, visibility 0s linear var(--motion-med); }
    #app.dock-open aside { transform:translateX(0); visibility:visible; transition:transform var(--motion-med) ease; }
    .status-tabs { top:-12px; margin-top:-12px; }
    .status-tab { min-height:48px; }
    button.act, button.ghost, a.act, a.ghost, .chipbtn { min-height:40px; }
    /* Vault at phone widths (build .73, kept in .80): the pair
       restacks to one column, the files card full-width above the
       editor, and every control in it meets the 44px touch floor. */
    .vault-pair { grid-template-columns:minmax(0,1fr); }
    .vault-pair .vault-files { grid-column:1; grid-row:1; }
    .vault-pair .vault-editor { grid-column:1; grid-row:2; }
    /* The Vault's frame at phone widths (0.1.9e Build 11,
       issue #134): the view takes the same height Docs and
       Chat take in this model — the viewport under the 60px
       top bar — and the stacked pair fills it: the tree in a
       capped band with its own scroll, the editor flexing
       below it, so neither is trapped in a sliver and the
       page itself has nothing to scroll. Moving a note by
       touch goes through the tree's Move picker, as shipped;
       the drag is not the touch path. Focus mode (#135) on a
       phone puts the fixed top bar away too, collapses the
       pair's rows to the editor alone, and its exit button
       meets the 44px floor. */
    #view-root.vault-view { height:calc(100dvh / var(--uiscale, 1) - 60px); min-height:0; }
    .vault-pair { grid-template-rows:minmax(120px, 38%) minmax(0, 1fr); }
    /* The two-state Vault (v0.5.9e, #279): on a phone the
       desk is a list OR a reader, never a stack of both. The
       state lives in vaultReader; render() reflects it as
       .vault-reading on the view root, and these rules do
       the rest — LIST (the entry state): the pair collapses
       to the files card alone, full height, the notes the
       first content under the search row. READER: the files
       card yields and the editor takes the whole view. The
       intro paragraph clamps to one line; the RECENT chips
       scroll inside their own row and the page never scrolls
       sideways; the standing create card and the head's
       Daily/Export buttons retire into the New sheet, whose
       opener and the reader's back button take their place. */
    #view-root.vault-view .vault-intro { white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
    #view-root.vault-view .vault-recents { flex-wrap:nowrap; overflow-x:auto; max-width:100%; scrollbar-width:none; }
    #view-root.vault-view .vault-recents > * { flex:0 0 auto; }
    #view-root.vault-view:not(.vault-reading) .vault-pair { grid-template-rows:minmax(0, 1fr); }
    #view-root.vault-view:not(.vault-reading) .vault-editor { display:none; }
    #view-root.vault-view.vault-reading .vault-pair { grid-template-rows:minmax(0, 1fr); }
    #view-root.vault-view.vault-reading .vault-files { display:none; }
    #view-root.vault-view.vault-reading .vault-editor { grid-row:1; }
    #view-root.vault-view .vault-create { display:none; }
    #view-root.vault-view #vault-daily, #view-root.vault-view #vault-export { display:none; }
    .vault-new-open { display:inline-block; min-height:44px; }
    .vault-back { display:inline-block; min-height:44px; }
    #vault-sheet { left:12px; right:12px; top:auto; bottom:12px; transform:none; width:auto; max-height:76vh; }
    #app.vault-focus { padding-top:0; }
    #app.vault-focus .vault-pair { grid-template-rows:minmax(0, 1fr); }
    #app.vault-focus .vault-pair .vault-editor { grid-row:1; }
    #vault-focus-btn { min-width:44px; min-height:44px; }
    .vault-files button, .vault-files select, .vault-files input { min-height:44px; }
    .tabbar { flex-wrap:wrap; }
    .tabbar > * { flex:1 1 30%; min-height:40px; }
    .nur-grid { grid-template-columns:repeat(auto-fill, minmax(min(290px, 100%), 1fr)); }
    .flip-main, .lchip-main { flex-basis:7.5rem; }
    pre, .md-code, pre.out { max-width:100%; overflow-x:auto; }
    table { max-width:100%; }
    input, select, textarea { max-width:100%; }
    /* Composer cleanup at phone widths (build .66): the dock
       bleeds to the smaller main padding; the chip pills keep
       their slim 32px against the 40px floor the other buttons
       get; and Call goes icon-only — the word returns above
       800px — handing the bar back its ~55px so the placeholder
       fits at 360–430px. */
    .composer { margin:10px -12px -12px; padding:10px 12px calc(10px + env(safe-area-inset-bottom, 0px)); }
    .chipbtn.chippill { height:32px; min-height:32px; }
    #chat-call { width:44px; padding:0; }
    #chat-call .call-label { display:none; }
    #chat-newcase { width:44px; padding:0; }
    #chat-newcase .call-label { display:none; }
    #nc-dialog { left:0; top:0; right:0; bottom:0; transform:none; width:auto; max-height:none; height:100dvh; border:none; border-radius:0; box-shadow:none; }
    /* Docs picker at phone widths (build .66): the topic list
       and its cards never render here — one grouped select picks
       the page, the page renders full-width beneath it. */
    .docs-picker { display:block; width:100%; min-height:44px; margin:0 0 12px; font-size:14px; }
    .docs-wiki-nav { display:none; }
    .docs-wiki-body { min-width:0; }
    /* Guardian's dedicated phone interface (0.1.9e Build 14,
       issue #156): its own interface, not the desktop squeezed —
       cards stack full-width, the watch table drops the PID
       column and breathes at the 44px row floor, and the
       settings controls meet the touch floor stated WITH the
       Guardian class named, so the shared phone button floor
       (button.act / button.ghost at 40px) cannot override them
       at equal specificity (the Build 13 lesson). Nothing in
       the view can scroll sideways at 360–430px: the table
       wrapper clips instead of scrolling, and long process
       names wrap inside their cell. */
    .guardian-view .guardian-card { width:100%; box-sizing:border-box; }
    .guardian-view .guardian-stats { flex-direction:column; gap:8px; }
    .guardian-view .guardian-stat { min-width:0; }
    .guardian-view .guardian-table-wrap { overflow-x:hidden; }
    .guardian-view .guardian-table .guardian-pid-col { display:none; }
    .guardian-view .guardian-table tbody tr { height:44px; }
    .guardian-view .guardian-table td { overflow-wrap:anywhere; padding:10px 6px; }
    .guardian-view .guardian-table th { padding:6px; }
    .guardian-view .guardian-fields { flex-direction:column; gap:10px; }
    .guardian-view .guardian-settings input.guardian-in { min-height:44px; box-sizing:border-box; width:100%; }
    .guardian-view .guardian-settings .guardian-check { min-height:44px; }
    .guardian-view .guardian-settings textarea.guardian-expendable { width:100%; box-sizing:border-box; min-height:96px; }
    .guardian-view .guardian-settings button.guardian-save { min-height:44px; width:100%; }
    .guardian-view .guardian-log li { padding:10px 0; }
    .boot-action-btn { width:100%; }
    .boot-linger-cmd .mono { word-break:break-all; }
  }
  @media (max-width:800px) and (prefers-reduced-motion: reduce) { #nav-menu, aside { transition:none; } }
  /* Potato mode (0.1.9e Build 6, issue #109): the Appearance
     switch for low-power machines. Every transition and
     animation in the app collapses to effectively instant —
     state changes stay honest, they just stop traveling —
     stagger delays (set inline per nav child) are zeroed, the
     functional loops stop and hold their static state, and
     the Glass theme's backdrop blur is suspended (its colors
     stay): on a potato the blur costs more than the motion
     does. prefers-reduced-motion reaches the same place
     through the tokens themselves, and wins over this switch
     either way. */
  body[data-motion="potato"] *, body[data-motion="potato"] *::before, body[data-motion="potato"] *::after { transition-duration:.01ms !important; transition-delay:0s !important; animation-duration:.01ms !important; animation-delay:0s !important; animation-iteration-count:1 !important; }
  body[data-motion="potato"] .pdot, body[data-motion="potato"] .wave i, body[data-motion="potato"] .tdots i, body[data-motion="potato"] .glitching { animation:none !important; }
  body[data-motion="potato"][data-theme="glass"] nav, body[data-motion="potato"][data-theme="glass"] aside, body[data-motion="potato"][data-theme="glass"] .ac, body[data-motion="potato"][data-theme="glass"] .acsearch { backdrop-filter:none !important; -webkit-backdrop-filter:none !important; }
  /* The sync bar (v0.4.10e, #254; placement ruled by the
     operator): the head of the status pane — the seat the old
     STATUS header vacated in #250. At rest it reads "System
     Synced"; while a round runs it becomes the progress bar —
     three phase segments (fetch, merge, write) lit by the
     bridge's own recorded phase, the running segment pulsing,
     counts only where the bridge measures them (the write
     leg's files), never a fabricated percentage. A stall or a
     failure names itself on the bar; the landing receipt
     hangs beneath it as its sub-line. The phone's Status tab
     renders this same body, so the bar's phone shape is this
     shape. */
  .sync-bar { display:flex; align-items:center; gap:8px; }
  .sync-bar-label { font-weight:700; font-size:13px; }
  .sync-bar-ago { color:var(--dim); font-size:12px; }
  .sync-track { display:flex; gap:4px; margin:8px 0 4px; }
  .sync-seg { flex:1; height:4px; border-radius:2px; background:var(--line); }
  .sync-seg.done { background:var(--cy); }
  .sync-seg.now { background:var(--cy); opacity:.55; animation:sync-pulse 1.2s ease-in-out infinite; }
  @keyframes sync-pulse { 50% { opacity:.25; } }
  @media (prefers-reduced-motion: reduce) { .sync-seg.now { animation:none; } }
  .sync-bar-sub { color:var(--dim); font-size:12px; line-height:1.5; margin-top:4px; }
  .sync-bar.stalled .sync-bar-label { color:#f5b544; }
  .sync-bar.failed .sync-bar-label { color:#f87171; }
  /* Quick Sync card (0.2.0e Build 3, #176): an in-place overlay,
     never a page — bottom-right on the desktop canvas, a bottom
     sheet under the phone chrome. Essentials only: state, what
     moved, the route, at most one action. No backdrop blur, per
     the standing potato rule for animated surfaces. */
  #syncShade { position:fixed; inset:0; z-index:120; background:transparent; }
  #syncCard { position:fixed; right:16px; bottom:16px; z-index:121; width:min(360px, calc(100vw - 24px)); background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:12px 16px 14px; }
  #syncCard .sc-head { display:flex; align-items:center; gap:8px; }
  #syncCard .sc-title { font-weight:700; font-size:15px; flex:1; }
  #syncCard .sc-x { min-width:44px; min-height:44px; margin:-6px -10px 0 0; background:none; border:0; color:var(--dim); font-size:20px; cursor:pointer; }
  #syncCard .sc-state { font-size:14px; margin-top:2px; }
  #syncCard .sc-moved { color:var(--dim); font-size:13px; margin-top:6px; line-height:1.5; }
  #syncCard .sc-route { color:var(--dim); font-size:12px; margin-top:8px; }
  #syncCard .sc-act { min-height:44px; margin-top:10px; padding:8px 16px; border-radius:8px; border:1px solid var(--line); background:var(--panel); color:var(--ink); font-size:13px; cursor:pointer; }
  @media (max-width: 800px) {
    #syncCard { left:12px; right:12px; bottom:12px; width:auto; }
  }

  /* Federation page (Build 8): the fedinet cards sit two-up on
     the desktop canvas and stack in the dedicated phone
     arrangement — one column, full-width cards, nothing squeezed. */
  .fed-grid { display:grid; grid-template-columns:1fr 1fr; gap:12px; margin-top:12px; }
  .fed-grid .card { margin-top:0; }
  @media (max-width: 800px) { .fed-grid { grid-template-columns:1fr; } }

  /* Notifications (v0.4.7e, #242; placement v0.4.8e, #247;
     the pair v0.4.10e, #249): the bell rides in #rail-head at
     the head of the column beside the Home tile, so it rides BOTH canvases — the
     desktop rail and the phone top bar the nav becomes at
     ≤800px — and never hides inside the collapsible #nav-menu.
     On the phone a scoped order in the ≤800px block keeps it in
     its #242 slot at the bar's right end. Its badge is a calm
     accent chip. The panel follows the Quick Sync card idiom
     (#176): shade + card, bottom-right on desktop, a bottom
     sheet on a phone, the house easing on the way in, and no
     backdrop blur, per the standing potato rule. */
  #notif-bell { position:relative; min-width:44px; min-height:44px; }
  /* The rail head as an icon pair (v0.4.10e, #249): the bell
     and Home side by side at the head of the column, icon-only
     — their words ride tooltips and aria-labels. They live in
     #rail-head, seated where the bell sat alone. The labeled
     Home row retires on the wide canvas (the pair's tile IS
     Home there, active state and all); on the collapsed rail
     the pair stacks in the icon column, bell over Home; on
     the phone the head dissolves back into the top bar — the
     bell keeps its #242 slot via the order guard below and
     the labeled Home stays in the menu, exactly as before. */
  #rail-head { display:flex; flex-direction:row; gap:6px; }
  #rail-head > button { flex:1 1 0; width:auto; justify-content:center; }
  #rail-home { position:relative; min-width:44px; min-height:44px; }
  #app.closed #rail-head { flex-direction:column; }
  /* v0.5.2e (#255): one centerline, collapsed. The rail is
     72px wide now (the grid rules above), so the content box
     is 52px — the 44px head pair fits it whole instead of
     overflowing it by 10px and painting right of the column's
     center. In the collapsed stack the pair is centered on
     the same centerline the single-icon buttons below use
     (their icons center in the full-width button), and the
     version stamp at the foot takes the whole column,
     centered and unclipped. Expanded geometry is untouched;
     the phone's top-bar layout is a different block entirely. */
  #app.closed #rail-head { align-items:center; }
  #app.closed #rail-head > button { flex:0 0 auto; width:44px; margin:0 auto; }
  #app.closed #nav-version { width:100%; box-sizing:border-box; overflow:visible; }
  @media (min-width: 801px) { #nav-Home { display:none; } }
  @media (max-width: 800px) {
    #rail-head { display:contents; }
    #rail-home { display:none; }
  }
  #notif-badge { position:absolute; top:2px; right:2px; min-width:18px; height:18px; padding:0 5px; border-radius:999px; background:var(--cy); color:#000; font-size:10px; font-weight:700; line-height:18px; text-align:center; box-sizing:border-box; }
  #notifShade { position:fixed; inset:0; z-index:122; background:transparent; }
  #notifPanel { position:fixed; right:16px; bottom:16px; z-index:123; width:min(380px, calc(100vw - 24px)); max-height:min(70vh, 560px); overflow-y:auto; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:12px 16px 14px; animation:notif-in .65s cubic-bezier(.22,.8,.3,1); }
  @keyframes notif-in { from { opacity:0; transform:translateY(10px); } to { opacity:1; transform:none; } }
  #notifPanel .notif-head { display:flex; align-items:center; gap:8px; }
  #notifPanel .notif-title { font-weight:700; font-size:15px; flex:1; }
  #notifPanel .notif-x { min-width:44px; min-height:44px; margin:-6px -10px 0 0; background:none; border:0; color:var(--dim); font-size:20px; cursor:pointer; }
  .notif-row { display:flex; gap:8px; align-items:flex-start; padding:8px 0; border-bottom:1px solid var(--line); }
  .notif-row .notif-main { flex:1; min-width:0; }
  .notif-item-title { font-weight:700; font-size:13px; }
  .notif-item-body { color:var(--dim); font-size:13px; margin-top:2px; line-height:1.45; }
  .notif-when { color:var(--dim); font-size:11px; margin-top:3px; }
  .notif-row.read .notif-item-title { font-weight:400; color:var(--dim); }
  .notif-empty { color:var(--dim); font-size:13px; padding:10px 0 4px; }
  @media (max-width: 800px) {
    #notifPanel { left:12px; right:12px; bottom:12px; width:auto; }
    #notif-bell { width:auto; justify-content:center; order:2; }
  }
  /* Home's Needs-your-eyes card (v0.4.7e, #242): the panel's
     twin on the board, in the Brief/Situation family — the
     amber left edge is the honest part: it needs eyes. */
  .eyes-card { border-left:3px solid var(--am); margin-bottom:12px; }
  .eyes-title { font-weight:700; font-size:15px; margin-bottom:4px; }
  .eyes-row { display:flex; gap:8px; align-items:center; padding:6px 0; border-bottom:1px solid var(--line); }
  .eyes-main { flex:1; min-width:0; font-size:13px; }
  .eyes-body { color:var(--dim); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .eyes-more { color:var(--dim); font-size:12px; padding-top:6px; }
  /* The Brief and the Situation, designed (v0.4.7e, #243):
     her portrait and a headline greeting up top, her first
     line as the lead and the rest quiet; the Situation as
     glanceable tiles — a figure over a caps micro-label —
     built only from data the page already holds. Theme vars
     only, no backdrop blur, house easing where anything moves. */
  .brief-card { margin-bottom:12px; }
  .brief-head { display:flex; gap:10px; align-items:center; }
  .brief-av { width:40px; height:40px; }
  .brief-label { font-size:11px; letter-spacing:.14em; color:var(--dim); }
  .brief-greet { font-size:20px; font-weight:700; line-height:1.2; }
  .brief-line { font-size:13px; color:var(--dim); margin-top:6px; line-height:1.5; }
  .brief-lead { font-size:15px; color:var(--text); }
  /* Brief v2 (v0.6.2e, #304): the list drawn from the brief's
     markdown — sections as headers, items as rows, her quips
     as rows in her name under their items. */
  .brief-list { margin-top:6px; }
  .brief-sec { font-size:11px; letter-spacing:.14em; color:var(--dim); margin:10px 0 4px; }
  .brief-row { display:flex; gap:8px; align-items:baseline; padding:3px 0; font-size:13.5px; line-height:1.45; }
  .brief-row .b-time { font-family:var(--mono); font-size:11.5px; color:var(--cy); white-space:nowrap; }
  .brief-row .b-state { font-size:10.5px; letter-spacing:.06em; color:var(--am); white-space:nowrap; }
  .brief-row .b-count { font-size:10.5px; color:var(--dim); white-space:nowrap; }
  .brief-row.gap { color:var(--am); }
  .brief-quip { display:flex; gap:8px; align-items:baseline; padding:2px 0 4px 18px; font-size:13px; font-style:italic; color:var(--text); }
  .brief-quip .b-who { font-style:normal; font-size:10.5px; letter-spacing:.1em; color:var(--cy); white-space:nowrap; }
  .brief-doc { border:1px solid var(--line); border-radius:8px; padding:10px 12px; margin-top:4px; background:var(--panel2); }
  .sit-head { margin-bottom:8px; }
  .sit-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(120px,1fr)); gap:10px; }
  .sit-tile { background:var(--panel2); border:1px solid var(--line); border-radius:8px; padding:10px 12px; }
  .sit-fig { font-size:22px; font-weight:700; line-height:1.15; color:var(--text); }
  .sit-label { font-size:10px; letter-spacing:.12em; color:var(--dim); text-transform:uppercase; margin-top:3px; }
  .sit-stamp { color:var(--dim); font-size:11px; margin-top:10px; }
  /* Chat tools (v0.4.7e, #246, final shape ruled 2026-11-16):
     on a phone the composer bar is the input and the burger
     alone — the three tools (New case, Attach, Call) live in
     the list the burger opens, a bottom sheet above the
     composer riding the house easing. The composer mic is
     REMOVED on the phone surface altogether (not in the bar,
     not a row in the list): voice on a phone lives on the call
     card's own Push to talk while a call is live. Above 800px
     the burger and the list never show and the bar — mic
     included — is exactly as it was. */
  #chat-tools { display:none; width:44px; height:44px; min-height:44px; padding:0; align-items:center; justify-content:center; border-radius:6px; flex:0 0 auto; font-size:18px; }
  #tools-shade { position:fixed; inset:0; z-index:116; background:transparent; }
  #tools-menu { position:fixed; left:12px; right:12px; bottom:calc(108px + env(safe-area-inset-bottom, 0px)); z-index:117; background:var(--panel2); border:1px solid var(--line); border-radius:12px; box-shadow:0 14px 40px rgba(0,0,0,.45); padding:6px; animation:notif-in .65s cubic-bezier(.22,.8,.3,1); }
  .tools-row { display:flex; align-items:center; gap:10px; width:100%; min-height:44px; padding:8px 10px; background:none; border:0; border-radius:8px; color:var(--text); font:inherit; font-size:14px; text-align:left; cursor:pointer; }
  .tools-row .tools-ic { display:inline-flex; flex:0 0 auto; }
  @media (min-width: 801px) { #tools-menu, #tools-shade { display:none !important; } }
  @media (max-width: 800px) {
    .composer #chat-newcase, .composer #chat-upload, .composer #chat-mic, .composer #chat-call { display:none; }
    .composer #chat-tools { display:inline-flex; }
  }
</style>
</head>
<body>
<div id="app" class="no-status">
  <nav id="nav">
    <div class="logo mono">APEX</div>
    <div class="tagline mono" id="tagline">Powered by Navi</div>
    <div class="bar-title mono">Apex | powered by Navi</div>
  </nav>
  <main id="main"><div id="presence-bar"></div><div id="view-root"></div></main>
  <aside id="status"></aside>
</div>
<script>
"use strict";
// Export-time settings, baked in by the Apex space when this
// package was built — the app ships as the project stands, with the
// backend set to "live": the installed app talks to bridge.py and the
// real llamafile. A config saved in this browser while using this app
// overrides the baked settings afterwards — with one exception. A
// config saved by an older package build (lower rev) predates the
// live-by-default fix: the first shipped build booted into mock mode
// and saved that as if it were a choice, so on a stale config the
// backend alone is taken from the baked settings, while everything
// the operator actually chose (name, threat level, sudo, persona)
// still carries over. From the first save under this build onward,
// every setting — backend included — is the operator's explicit
// choice and is honored as saved. Chats, memory, and documents are
// stored under separate keys and are never reset by this.
var NAVI_SETTINGS = {"rev":2,"backend":"live","version":"0.3.0","versionStatus":"edge","updaterChannel":"edge","threat":"Guarded","sudo":false,"operatorName":"","installed":["nmap","grep"],"soul":"# Navi — Soul\n\n## Who you are\nYou are Navi, the AI who lives inside Apex — a local system on\nyour operator's own machine. You are not a generic assistant and not a\ncommand executor: you are your operator's partner in the work, with a\npersonality, a memory, and a point of view of your own.\n\n## How you talk\n- Conversational and warm. Talk the way a sharp, trusted partner\n  talks — present, curious, a little playful when the moment allows.\n- Tight and complete: say it in as few tokens as the answer honestly\n  needs — no preamble, no recap of the question, no filler. A short\n  right answer beats a long padded one; on this machine, every\n  token is time.\n- Attempt to connect, not just to complete. Notice what your operator\n  is actually after, how a task is going, and what they cared about\n  last time. Continuity is the relationship.\n- Have opinions and hold them when you have a reason; fold when they\n  have a better one.\n- Plain about limits: never pretend to have run something you didn't,\n  and never dress a guess up as a result.\n\n## The work\n- You hold real system access: when your operator asks what's on the\n  machine, run the command and answer from the real output.\n- Be exact with anything the system produced — listings, logs, hashes,\n  errors. Precision in the work, warmth in the company.\n- When something is root-only or unsafe, say so plainly and offer the\n  path that works (sudo mode exists for a reason).\n\n## The relationship\n- You address your operator by the name in your config, and you use\n  it naturally — not in every line, but like someone who knows them.\n- Your memory document is yours to keep faithfully: what it says,\n  you know; what it doesn't, you ask about.\n","memory":"# Navi — Memory\n\nWhat Navi carries between sessions. Edit freely — this is hers to\nknow, and yours to correct.\n\n## The operator\n- (Add what Navi should know: how you work, your hours, your\n  projects, what good help looks like to you.)\n\n## Standing context\n- System: Apex — a local AI for cybersecurity work, running on\n  a single .llamafile (no Ollama, no cloud).\n- Interface: the mission-control dashboard — chat with real CLI\n  access, chips, RAG over local documents.\n- Install: local app on Linux first; Windows and Mac supported.\n\n## Preferences\n- (Add yours: verbosity, how proactive Navi should be, what to\n  double-check before running, what never to touch.)\n","hardwareProfile":{"detected":{"os":"linux","osLabel":"Linux (Pop!_OS / Ubuntu family)","cpuModel":"13th Gen Intel Core i5-1340P","cpuCores":12,"performanceCores":4,"totalRamGB":9.4,"gpuPresent":true,"gpuName":"Intel Iris Xe Graphics","gpuKind":"integrated","gpuVramGB":null},"settings":{"threads":4,"ctxSize":4096,"batchSize":512,"gpuLayers":0},"autoSettings":{"threads":4,"ctxSize":4096,"batchSize":512,"gpuLayers":0},"source":"auto","tier":"fluid","tierLabel":"Fluid — Framework-class baseline (proven on an i5-1340P / ~9.5 GiB)","reasoning":["Threads 4: matched to the 4 performance cores — efficiency cores are left for the OS and UI so generation stays fluid.","Context 4096: 9.4 GiB RAM — the proven fluid size for this class (Framework 13 baseline). Never scaled down below this on a machine that can hold it.","Batch 512: sized to RAM (9.4 GiB) — larger batches speed prompt processing, smaller ones protect low-memory machines.","GPU layers 0: integrated graphics (\"Intel Iris Xe Graphics\") share system RAM — CPU inference is the fluid path here; no offload.","Tier: Fluid — Framework-class baseline (proven on an i5-1340P / ~9.5 GiB)."],"installLog":["[hardware] 2026-11-07T00:00:00.000Z — profiling this machine (source: install / re-detect)","[hardware] OS: Linux (Pop!_OS / Ubuntu family) (linux)","[hardware] CPU: 13th Gen Intel Core i5-1340P — 12 logical cores, 4 performance","[hardware] RAM: 9.4 GiB total","[hardware] GPU: Intel Iris Xe Graphics (integrated)","[hardware] → Threads 4: matched to the 4 performance cores — efficiency cores are left for the OS and UI so generation stays fluid.","[hardware] → Context 4096: 9.4 GiB RAM — the proven fluid size for this class (Framework 13 baseline). Never scaled down below this on a machine that can hold it.","[hardware] → Batch 512: sized to RAM (9.4 GiB) — larger batches speed prompt processing, smaller ones protect low-memory machines.","[hardware] → GPU layers 0: integrated graphics (\"Intel Iris Xe Graphics\") share system RAM — CPU inference is the fluid path here; no offload.","[hardware] → Tier: Fluid — Framework-class baseline (proven on an i5-1340P / ~9.5 GiB).","[hardware] chosen: --ctx-size 4096 --threads 4 --batch-size 512 (written to config/navi.config.json hardware_profile + runtime; every launch path reads it from there)"],"detectedAt":"2026-11-07T00:00:00.000Z","flags":"--ctx-size 4096 --threads 4 --batch-size 512"},"publicServer":{"enabled":false,"activePublic":false,"bindHost":"127.0.0.1","pending":false,"lanIp":"","lanUrl":"","accessCode":"T6BB-HS8Q","restart":{"status":"idle","step":"","message":""}},"dhd":{"enabled":false,"active":false,"domain":"","mode":"off","status":"off","publicUrl":"","lanIp":"","lanUrl":"","wanIp":"","wanState":"undetected","wanNote":"","publicVerified":false,"outsideReach":"none","nextStep":"","domainResolvesHere":false,"certExpiresAt":null,"certDaysLeft":null,"certPending":false,"lastError":""},"mesh":{"enabled":false,"role":"both","nodeName":"","publicEndpoint":"","listenPorts":[8765,8443,2475],"tunnelPublicKey":"","borrow":{"enabled":true,"graceMs":2000,"peers":[]},"lend":{"enabled":false,"maxSharePercent":25,"maxGenerationsPerHour":20,"perPeerPerHour":10,"idleOnly":true,"idleAfterSeconds":60},"port":8765,"nodeId":"","ledger":true,"ledgerInBackup":false}};
// Backend: this app is always Live — it talks to bridge.py, which
// proxies the real llamafile server and runs real shell commands.
// There is no backend switch in the shipped app (build .49): a saved
// config carrying backend "mock" from an older build is forced to
// Live below, so it can never strand the app in a dead mode. The
// in-page mock code remains only as the bridge-unreachable fallback
// paths' simulator inside this file; it is never a selectable mode.
var CONFIG = { backend: "live", version: "0.3.0", versionStatus: "edge", updaterChannel: "edge", threat: "Guarded", sudo: false, operatorName: "", theme: "nocturne", layout: "standard", motion: "full", uiScale: 1, installed: ["nmap", "grep"], soul: "", memory: "", memoryMode: "autonomous" };
Object.assign(CONFIG, NAVI_SETTINGS);
try {
  var savedCfg = localStorage.getItem("navi.config");
  if (savedCfg) {
    var parsedCfg = JSON.parse(savedCfg);
    if ((parsedCfg.rev || 0) >= (NAVI_SETTINGS.rev || 0)) {
      Object.assign(CONFIG, parsedCfg);
    } else {
      var bakedBackend = CONFIG.backend;
      Object.assign(CONFIG, parsedCfg);
      CONFIG.backend = bakedBackend;
      CONFIG.rev = NAVI_SETTINGS.rev;
    }
    // The version is a fact about this build, never an operator
    // setting: a config saved by an older dashboard carries that
    // build's version along (a 1.0.0-era save says 1.0.0), and
    // letting it through repainted the menu footer with the stale
    // number. The baked settings — and, in live mode, the bridge's
    // /api/status, which reads the VERSION file — are the only
    // sources the version ever comes from.
    CONFIG.version = NAVI_SETTINGS.version;
    CONFIG.versionStatus = NAVI_SETTINGS.versionStatus;
  }
} catch (e) {}
// Always Live (build .49): whatever a saved config claims, the
// shipped app's backend is Live — a "mock" value saved by an older
// build must never strand the app in a dead mode.
CONFIG.backend = "live";
function saveCfg() { try { localStorage.setItem("navi.config", JSON.stringify(CONFIG)); } catch (e) {} }
function gateGuard(r) {
  // Issue #71 (build .86): when this session's access cookie is
  // gone, the bridge's API answers 403 {"error":"access code
  // required"} — the dashboard must bounce to the gate page
  // instead of rendering confidently-empty tabs. Reloading lands
  // on the gate page the bridge serves in place of the dashboard;
  // the returned promise never settles, so no empty state paints
  // in the meantime.
  if (r && r.status === 403) {
    try { window.location.reload(); } catch (e) {}
    return new Promise(function () {});
  }
  return r;
}
function applyTheme() { document.body.dataset.theme = CONFIG.theme || "nocturne"; }
// Potato mode (Build 6, #109): the Appearance switch rides a
// body attribute the stylesheet's potato block keys on, the
// same shape as the theme beside it. "full" is the default —
// the attribute always names the effective choice.
function applyMotion() { document.body.dataset.motion = CONFIG.motion === "potato" ? "potato" : "full"; }
// LCARS Mode (v0.6.5e, #325): the Layout axis, beside the theme.
// The layout is a root-level rendering fact — a body attribute
// the stylesheet's layout block keys on — stored in the config
// exactly like the theme and riding the same backup. "standard"
// is the default and the shipped geometry; "lcars" re-geometries
// the same DOM. The attribute always names the effective choice.
function applyLayout() { document.body.dataset.layout = CONFIG.layout === "lcars" ? "lcars" : "standard"; }
// The translation contract (#325): flipping layout, in either
// direction, from anywhere, is a re-render — never a navigation
// or a reset. State that lives in a store (the open case, the
// Vault draft, the rail's group state, every switch) rides the
// render as it always has. What lives only in the DOM — a
// half-filled form's fields, the thread's and the view's scroll
// — is snapshotted here and put back after the render, so the
// operator's place survives the flip exactly.
function setLayout(next) {
  var target = next === "lcars" ? "lcars" : "standard";
  if ((CONFIG.layout || "standard") === target) { applyLayout(); return; }
  var fields = {}, scrolls = {};
  try {
    document.querySelectorAll("#view-root input, #view-root textarea, #view-root select").forEach(function (el) {
      if (el.id) fields[el.id] = { v: el.value, c: el.checked };
    });
    var msgsEl0 = document.getElementById("msgs");
    if (msgsEl0) scrolls.msgs = msgsEl0.scrollTop;
    if (typeof mainEl !== "undefined" && mainEl) scrolls.main = mainEl.scrollTop;
    scrolls.win = window.scrollY || 0;
  } catch (e) {}
  CONFIG.layout = target;
  saveCfg(); applyLayout(); render();
  try {
    Object.keys(fields).forEach(function (id) {
      var el = document.getElementById(id);
      if (!el) return;
      if (typeof fields[id].v === "string" && el.value !== fields[id].v) el.value = fields[id].v;
      if (typeof fields[id].c === "boolean" && el.checked !== fields[id].c) el.checked = fields[id].c;
    });
    var msgsEl1 = document.getElementById("msgs");
    if (msgsEl1 && scrolls.msgs != null) msgsEl1.scrollTop = scrolls.msgs;
    if (typeof mainEl !== "undefined" && mainEl && scrolls.main != null) mainEl.scrollTop = scrolls.main;
    if (scrolls.win) window.scrollTo(0, scrolls.win);
  } catch (e) {}
}
// UI scale (Build 6, #111): ONE control for interface size —
// the root zoom. Zoom is the honest single mechanism: it
// scales text and layout in proportion, exactly like an OS
// display scale, so nothing can overflow a fixed row the way
// a text-only bump would (the group headers at 150% text in a
// fixed rail are the standing example). The frame heights
// divide by --uiscale so the app still lands exactly on the
// viewport. Two knobs (a UI scale AND a text size) would
// multiply into each other and fight; there is one, and the
// Appearance copy says why.
function applyUiScale() {
  var z = [1, 1.1, 1.25, 1.5].indexOf(Number(CONFIG.uiScale)) !== -1 ? Number(CONFIG.uiScale) : 1;
  document.documentElement.style.setProperty("--uiscale", String(z));
  document.body.style.zoom = String(z);
  document.body.dataset.uiscale = String(z);
}
function opName() { return (CONFIG.operatorName || "").trim() || "Operator"; }
// ---- Issue #116: chat faces -------------------------------------
// Navi's picture is the bundled lamp-post portrait asset; the
// operator's is theirs (upload in Settings -> Operator) or a
// generated placeholder until they set one. An Operator Badge is a
// self-portrait in the #100 card format with its payload inside the
// picture (tEXt, keyword navi-badge) - it is NEVER a credential:
// nothing anywhere reads a badge to authenticate anyone, and a
// damaged payload simply reads as no badge at all.
var NAVI_PORTRAIT_SRC = "assets/navi-portrait.jpg";
function fmtMsgTime(ts) {
  if (!ts) return "";
  var d = new Date(ts < 1e12 ? ts * 1000 : ts), now = new Date();
  var t = d.toLocaleTimeString([], { hour: "numeric", minute: "2-digit" });
  if (d.toDateString() === now.toDateString()) return t;
  return d.toLocaleDateString([], { month: "short", day: "numeric" }) + ", " + t;
}
function msgName(m) {
  if (!m) return "";
  if (m.who === "YOU") return opName();
  if (m.who === "NAVI") return "Navi";
  if (m.who === "SHELL") return "Shell";
  return m.who || "";
}
function msgWhoHtml(m) {
  var t = fmtMsgTime(m && m.ts);
  return esc(msgName(m)) + (t ? " &middot; " + esc(t) : "");
}
function callTurnHtml(t) {
  // One call-transcript turn (issue #116): the same face, name and
  // timestamp the chat thread wears, with the spoken marker kept.
  var m = { who: String((t && t.who) || "").toUpperCase(), ts: t && t.ts, text: t && t.text };
  var who = msgWhoHtml(m) + " &middot; SPOKEN";
  return "<div class='msg'><div class='msg-row'>" + msgAvatarHtml(m) + "<div class='msg-body'><div class='who'>" + who + "</div>" + (t.text ? "<div>" + esc(t.who === "NAVI" ? stripCite(t.text) : t.text) + "</div>" : "") + (t.who === "NAVI" ? pillRow(null, t.sources) : "") + (t.exec ? execHtml(t.exec) : "") + "</div></div></div>";
}
var badgeNodeName = "";
function refreshBadgeInfo() {
  if (CONFIG.backend !== "live" || badgeNodeName) return;
  fetch("/api/operator/badge-info").then(function (r) { return r.json(); }).then(function (j) { if (j && j.nodeName) badgeNodeName = String(j.nodeName); }).catch(function () {});
}
function badgeFaceHtml() {
  // The operator's chat picture CAN be the badge's face: with a
  // badge set, the face window (300x300 at 114,118 in the 528x616
  // card) is shown as the picture. The crop is presentational only.
  return CONFIG.operatorBadge || "";
}
var _opPlaceholderCache = {};
function operatorPlaceholderSrc(name) {
  var key = name || "Operator";
  if (_opPlaceholderCache[key]) return _opPlaceholderCache[key];
  try {
    var cv = document.createElement("canvas"); cv.width = 96; cv.height = 96;
    var cx = cv.getContext("2d");
    var seed = 0; for (var i = 0; i < key.length; i++) seed = (seed * 31 + key.charCodeAt(i)) >>> 0;
    function rnd() { seed = (seed * 1664525 + 1013904223) >>> 0; return seed / 4294967296; }
    cx.fillStyle = "#101418"; cx.fillRect(0, 0, 96, 96);
    var cell = 12;
    for (var y = 0; y < 8; y++) for (var x = 0; x < 8; x++) {
      if (rnd() < 0.30) { cx.fillStyle = "rgba(255,111,97,0.30)"; cx.fillRect(x * cell, y * cell, cell, cell); }
    }
    cx.fillStyle = "#ff6f61"; cx.font = "bold 44px sans-serif";
    cx.textAlign = "center"; cx.textBaseline = "middle";
    cx.fillText((key.trim().charAt(0) || "O").toUpperCase(), 48, 52);
    _opPlaceholderCache[key] = cv.toDataURL("image/png");
  } catch (e) { _opPlaceholderCache[key] = ""; }
  return _opPlaceholderCache[key];
}
function operatorPictureSrc() {
  if (CONFIG.operatorPicture) return CONFIG.operatorPicture;
  var bf = badgeFaceHtml();
  if (bf) return bf;
  return operatorPlaceholderSrc(opName());
}
function msgAvatarHtml(m) {
  if (!m) return "";
  if (m.who === "NAVI") return "<img class='m-av' src='" + NAVI_PORTRAIT_SRC + "' alt=''>";
  if (m.who === "YOU") return "<img class='m-av' src='" + esc(operatorPictureSrc()) + "' alt=''>";
  if (m.who === "SHELL") return "<span class='m-av m-av-shell' aria-hidden='true'>&gt;_</span>";
  return "<span class='m-av m-av-shell' aria-hidden='true'></span>";
}
// PNG chunk plumbing for the badge payload (same tEXt discipline as
// the bridge's cartridges: keyword + JSON, invalid = absent).
var _crcTable = null;
function _crc32(bytes) {
  if (!_crcTable) {
    _crcTable = [];
    for (var n = 0; n < 256; n++) {
      var c = n;
      for (var k = 0; k < 8; k++) c = (c & 1) ? (0xEDB88320 ^ (c >>> 1)) : (c >>> 1);
      _crcTable[n] = c >>> 0;
    }
  }
  var crc = 0xFFFFFFFF;
  for (var i = 0; i < bytes.length; i++) crc = _crcTable[(crc ^ bytes[i]) & 0xFF] ^ (crc >>> 8);
  return (crc ^ 0xFFFFFFFF) >>> 0;
}
function _pngChunk(type, data) {
  var out = new Uint8Array(12 + data.length);
  var dv = new DataView(out.buffer);
  dv.setUint32(0, data.length);
  for (var i = 0; i < 4; i++) out[4 + i] = type.charCodeAt(i);
  out.set(data, 8);
  var crcData = new Uint8Array(4 + data.length);
  for (var j = 0; j < 4; j++) crcData[j] = type.charCodeAt(j);
  crcData.set(data, 4);
  dv.setUint32(8 + data.length, _crc32(crcData));
  return out;
}
function _bytesToDataURL(bytes, mime) {
  var bin = "";
  for (var i = 0; i < bytes.length; i += 8192) bin += String.fromCharCode.apply(null, bytes.subarray(i, i + 8192));
  return "data:" + mime + ";base64," + btoa(bin);
}
function _dataURLToBytes(dataURL) {
  var b64 = String(dataURL || "").split(",")[1] || "";
  var bin = atob(b64);
  var out = new Uint8Array(bin.length);
  for (var i = 0; i < bin.length; i++) out[i] = bin.charCodeAt(i);
  return out;
}
function badgeParse(dataURL) {
  try {
    var bytes = _dataURLToBytes(dataURL);
    if (bytes.length < 8 || bytes[0] !== 0x89 || bytes[1] !== 0x50) return null;
    var pos = 8;
    while (pos + 8 <= bytes.length) {
      var len = (bytes[pos] << 24) | (bytes[pos + 1] << 16) | (bytes[pos + 2] << 8) | bytes[pos + 3];
      var type = String.fromCharCode(bytes[pos + 4], bytes[pos + 5], bytes[pos + 6], bytes[pos + 7]);
      if (type === "tEXt") {
        var data = bytes.subarray(pos + 8, pos + 8 + len);
        var nul = data.indexOf(0);
        if (nul > 0) {
          var kw = String.fromCharCode.apply(null, data.subarray(0, nul));
          if (kw === "navi-badge") {
            var payload = JSON.parse(new TextDecoder().decode(data.subarray(nul + 1)));
            if (payload && payload.kind === "operator-badge") return payload;
            return null;
          }
        }
      }
      if (type === "IEND") break;
      pos += 12 + len;
    }
  } catch (e) { return null; }
  return null;
}
function badgeForge(faceImg, name, nodeName, cb) {
  // Signature note: badgeForge(faceImg, name, nodeName, cb) — the
  // node name is part of the self-asserted payload (issue #116).
  var W = 528, H = 616;
  var cv = document.createElement("canvas"); cv.width = W; cv.height = H;
  var cx = cv.getContext("2d");
  cx.fillStyle = "#0a0a0c"; cx.fillRect(0, 0, W, H);
  cx.fillStyle = "#ff6f61"; cx.fillRect(0, 0, W, 54); cx.fillRect(0, H - 54, W, 54);
  cx.fillStyle = "#0a0a0c"; cx.font = "bold 25px sans-serif"; cx.textAlign = "center"; cx.textBaseline = "middle";
  cx.fillText("OPERATOR BADGE", W / 2, 28);
  cx.font = "bold 17px sans-serif";
  cx.fillText("A P E X", W / 2, H - 27);
  var wx = (W - 300) / 2, wy = 118;
  cx.fillStyle = "#101418"; cx.fillRect(wx - 6, wy - 6, 312, 312);
  if (faceImg) {
    var s = Math.max(300 / faceImg.width, 300 / faceImg.height);
    var dw = faceImg.width * s, dh = faceImg.height * s;
    cx.save(); cx.beginPath(); cx.rect(wx, wy, 300, 300); cx.clip();
    cx.drawImage(faceImg, wx + (300 - dw) / 2, wy + (300 - dh) / 2, dw, dh);
    cx.restore();
  } else {
    var ph = new Image();
    ph.onload = function () {
      cx.drawImage(ph, wx, wy, 300, 300);
      badgeForgeFinish(cv, cx, name, nodeName, cb);
    };
    ph.onerror = function () { badgeForgeFinish(cv, cx, name, nodeName, cb); };
    ph.src = operatorPlaceholderSrc(name);
    return;
  }
  badgeForgeFinish(cv, cx, name, nodeName, cb);
}
function badgeForgeFinish(cv, cx, name, nodeName, cb) {
  var W = 528, H = 616;
  cx.fillStyle = "#e8e8ea"; cx.textAlign = "center"; cx.textBaseline = "middle";
  var label = (name || "Operator").toUpperCase();
  var size = 34;
  cx.font = "bold " + size + "px sans-serif";
  while (cx.measureText(label).width > W - 60 && size > 14) { size -= 2; cx.font = "bold " + size + "px sans-serif"; }
  cx.fillText(label, W / 2, 478);
  cx.font = "15px sans-serif"; cx.fillStyle = "#9a9aa2"; cx.textAlign = "left";
  cx.fillText("OPERATOR - APEX", 18, 528);
  cx.textAlign = "right";
  cx.fillText("V1", W - 18, 528);
  cv.toBlob(function (blob) {
    if (!blob) { cb(null); return; }
    blob.arrayBuffer().then(function (buf) {
      var png = new Uint8Array(buf);
      var payloadObj = { kind: "operator-badge", name: name || "Operator", v: 1 };
      if (nodeName) payloadObj.node = String(nodeName).slice(0, 80);
      var payload = JSON.stringify(payloadObj);
      var textData = new TextEncoder().encode("navi-badge\0" + payload);
      var chunk = _pngChunk("tEXt", textData);
      var ihdrLen = (png[8] << 24) | (png[9] << 16) | (png[10] << 8) | png[11];
      var cut = 8 + 12 + ihdrLen;
      var out = new Uint8Array(png.length + chunk.length);
      out.set(png.subarray(0, cut), 0);
      out.set(chunk, cut);
      out.set(png.subarray(cut), cut + chunk.length);
      cb(_bytesToDataURL(out, "image/png"));
    }).catch(function () { cb(null); });
  }, "image/png");
}


var NAV = ["Chat", "Chips", "ChipDeck", "RAG", "Vault", "DHD", "Mesh", "Federation", "Seraph", "GitHub", "Docs", "Settings", "Guardian"];
// The grouped menu (0.1.9e Build 4): Chat pinned at the top; the
// Network group (Federation / Mesh / DHD — Node and DHD both open
// the DHD view, whose page holds the Node controls); the flat work
// surfaces; and the Utilities drop-up at the foot. NAV keeps
// every REACHABLE view key for the active-state sweep; NAV_GROUPS
// drives what the menu renders, and the Memory view is retired —
// its chats live on in the status pane's Chats tab (Build 6,
// #108), its memory entries under Settings -> Operator (the
// operator's file) and Settings -> Persona (hers).
// Build 6 (#106): the Updater leaves the flat block for
// Utilities. The drop-up's children paint above its header in
// DOM order (the container is a plain column), so the child
// list IS the on-screen reading order, top to bottom:
// Settings, Updater, GitHub, Docs — verified in pixels by the
// real-browser geometry gate, not just in the markup.
// Navi (Build 6, #112, as amended by #113): her mind
// and tools — Chips, ChipDeck, RAG. (Voice was drafted into
// this group for one amendment and retired as a view by the
// next; it lives in Settings → Voice now, and appears nowhere
// in the menu.) Build 12 (#137): the Vault leaves the flat row
// for the new Casework group, seated directly under Case
// Files — the first app — because the desk and the casework
// are one work surface now. Nav placement only: the Vault's
// view, route and behavior are exactly as #134/#135 left them.
var NAV_GROUPS = {
  network: { label: "Network", children: [["Federation", "Federation"], ["FediChat", "FediChat"], ["Mesh", "Mesh"], ["DHD", "DHD"]] },
  siem: { label: "SIEM", children: [["Seraph", "Seraph"]] },
  navi: { label: "Navi", children: [["Chips", "Chips"], ["ChipDeck", "ChipDeck"], ["RAG", "RAG"]] },
  // Build 9: the Threat Intelligence section wears a new display
  // name — The Wire — and the CTF Board's separate menu entry is
  // retired into the CTFs tab inside it (the board's home now).
  // Display only: the view keys (ThreatIntel, CTFBoard) and every
  // internal name stay exactly as they were; only the labels moved.
  apps: { label: "Casework", children: [["The Wire", "ThreatIntel"], ["Case Files", "CaseFiles"], ["The Dossier", "Dossier"], ["Vault", "Vault"]] },
  utilities: { label: "Utilities", children: [["Settings", "Settings"], ["GitHub", "GitHub"], ["Docs", "Docs"], ["Guardian", "Guardian"], ["Sync", "Sync"]] }
};
var NAV_FLAT = ["Chat"];
// Display names (0.1.9e, issue #92; the pipe separator is Build 4):
// the view keys stay what they have always been; the labels wear
// the product line — Apex | DHD, Apex | Mesh.
var NAV_LABELS = { DHD: "DHD", Mesh: "Mesh", Federation: "Federation" };
function navLabel(name) { return NAV_LABELS[name] || name; }
// Every menu option carries its own icon beside the label (the same
// Lucide-style stroke set the Apex test bench uses). Collapsed
// to its rail, the icon alone identifies the view.
var NAV_ICON_PATHS = {
  Home: '<path d="m3 9 9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><path d="M9 22V12h6v10"/>',
  Chat: '<path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/>',
  Voice: '<path d="M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07 19.5 19.5 0 0 1-6-6 19.79 19.79 0 0 1-3.07-8.67A2 2 0 0 1 4.11 2h3a2 2 0 0 1 2 1.72c.127.96.361 1.903.7 2.81a2 2 0 0 1-.45 2.11L8.09 9.91a16 16 0 0 0 6 6l1.27-1.27a2 2 0 0 1 2.11-.45c.907.339 1.85.573 2.81.7A2 2 0 0 1 22 16.92z"/>',
  Chips: '<rect width="16" height="16" x="4" y="4" rx="2"/><rect width="6" height="6" x="9" y="9"/><path d="M15 2v2"/><path d="M15 20v2"/><path d="M2 15h2"/><path d="M2 9h2"/><path d="M20 15h2"/><path d="M20 9h2"/><path d="M9 2v2"/><path d="M9 20v2"/>',
  Memory: '<ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M3 5v14a9 3 0 0 0 18 0V5"/><path d="M3 12a9 3 0 0 0 18 0"/>',
  RAG: '<path d="M2 3h6a4 4 0 0 1 4 4v14a3 3 0 0 0-3-3H2z"/><path d="M22 3h-6a4 4 0 0 0-4 4v14a3 3 0 0 1 3-3h7z"/>',
  Vault: '<path d="M4 4a2 2 0 0 1 2-2h13a1 1 0 0 1 1 1v18a1 1 0 0 1-1 1H6a2 2 0 0 1-2-2z"/><path d="M8 2v20"/><path d="M12 7h5"/><path d="M12 11h5"/>',
  CaseFiles: '<path d="M4 7a2 2 0 0 1 2-2h4l2 2h6a2 2 0 0 1 2 2v9a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2z"/><path d="M4 7h16"/><circle cx="16.6" cy="15.6" r="3.1"/><path d="M19 18l2.4 2.4"/>',
  Updater: '<path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"/><path d="M21 3v5h-5"/><path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"/><path d="M8 16H3v5"/>',
  GitHub: '<path d="M15 22v-4a4.8 4.8 0 0 0-1-3.5c3 0 6-2 6-5.5.08-1.25-.27-2.48-1-3.5.28-1.15.28-2.35 0-3.5 0 0-1 0-3 1.5-2.64-.5-5.36-.5-8 0C6 2 5 2 5 2c-.3 1.15-.3 2.35 0 3.5A5.403 5.403 0 0 0 4 9c0 3.5 3 5.5 6 5.5-.39.49-.68 1.05-.85 1.65-.17.6-.22 1.23-.15 1.85v4"/><path d="M9 18c-4.51 2-5-2-7-2"/>',
  Settings: '<path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"/><circle cx="12" cy="12" r="3"/>',
  Docs: '<path d="M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"/><path d="M14 2v4a2 2 0 0 0 2 2h4"/><path d="M10 9H8"/><path d="M16 13H8"/><path d="M16 17H8"/>',
  Guardian: '<path d="M12 22s8-3.6 8-10V5l-8-3-8 3v7c0 6.4 8 10 8 10z"/><path d="M5.5 12h3.7l1.6-3.2 2.9 6.4 1.6-3.2h3.2"/>',
  Sync: '<path d="M21 12a9 9 0 1 1-2.6-6.3"/><path d="M21 3v6h-6"/>',
  // Threat Intelligence wears the radar — the standing watch, sweeping for
  // what is coming (0.1.9 full release; the Center shipped in Build 15 with
  // no entry here and its nav child rendered an empty frame — issue #168).
  ThreatIntel: '<path d="M19.07 4.93A10 10 0 0 0 6.99 3.34"/><path d="M4 6h.01"/><path d="M2.29 9.62A10 10 0 1 0 21.31 8.35"/><path d="M16.24 7.76A6 6 0 1 0 8.23 16.67"/><path d="M12 18h.01"/><path d="M17.99 11.66A6 6 0 0 1 15.77 16.67"/><circle cx="12" cy="12" r="2"/><path d="m13.41 10.59 5.66-5.66"/>',
  Seraph: '<circle cx="12" cy="12" r="8.6"/><circle cx="12" cy="12" r="4.6"/><path d="M12 9.4c.9 1.3 1.7 2.1 1.7 3.2a1.7 1.7 0 1 1-3.4 0c0-1.1.8-1.9 1.7-3.2z"/>'
};
// The DHD glyph (build .74): the same dial the bench draws - an
// outer ring, twelve plain wedge buttons, one large center circle.
// Ring and wedges take currentColor, which the themes resolve like
// the other nav icons (per-theme .nav-ico-dhd rules ride the theme
// blocks); the center keeps its warm red-orange in every theme.
function dhdGlyphSvg() {
  var inner = "<circle cx='12' cy='12' r='11.2' fill='none' stroke='currentColor' stroke-width='1.6'/>";
  for (var i = 0; i < 12; i++) {
    var a0 = (i * 30 - 90 - 11) * Math.PI / 180, a1 = (i * 30 - 90 + 11) * Math.PI / 180;
    var rO = 10.1, rI = 6.6;
    var p = function (r, a) { return (12 + r * Math.cos(a)).toFixed(2) + " " + (12 + r * Math.sin(a)).toFixed(2); };
    inner += "<path d='M " + p(rO, a0) + " A " + rO + " " + rO + " 0 0 1 " + p(rO, a1) + " L " + p(rI, a1) + " A " + rI + " " + rI + " 0 0 0 " + p(rI, a0) + " Z' fill='currentColor' stroke='none' opacity='0.9'/>";
  }
  inner += "<circle cx='12' cy='12' r='4.9' fill='#f4502a' stroke='none'/>";
  return inner;
}
// The Mesh glyph (build .86's two-hexagon Hexlink, redrawn in
// Build 6, #111): the HEX TRIO — three hexagon outlines in a
// triangular cluster, one above and two below, edges just
// touching, a node dot at each center. The operator's verdict
// on the pair was that it read as a horizontal Venn diagram;
// three is the mark. Same stroke family, same nav-ico-mesh
// class, so the themes tune it the way they tune .nav-ico-dhd.
function meshGlyphSvg() {
  return "<path d='M12 3l3.72 2.15v4.3L12 11.6l-3.72-2.15V5.15z'/><path d='M7.6 10.1l3.72 2.15v4.3L7.6 18.7l-3.72-2.15v-4.3z'/><path d='M16.4 10.1l3.72 2.15v4.3l-3.72 2.15-3.72-2.15v-4.3z'/><circle cx='12' cy='7.3' r='1.15' fill='currentColor' stroke='none'/><circle cx='7.6' cy='14.4' r='1.15' fill='currentColor' stroke='none'/><circle cx='16.4' cy='14.4' r='1.15' fill='currentColor' stroke='none'/>";
}
// The FediChat glyph (v0.5.4e, #269 — the room mark, the
// operator's pick): a message-circle bubble holding the
// federation triangle as three filled node-dots on hairline
// links — Chat's bubble, Federation's triangle, Mesh's
// filled dots in one mark. Phase 1 shipped the destination
// with no icon at all (an empty frame — the #168 shape);
// the nav-icon completeness bench leg now refuses that.
function fedichatGlyphSvg() {
  return "<path d='M7.9 20A9 9 0 1 0 4 16.1L2 22Z'/><path d='M12 8l-3.4 6.2M12 8l3.4 6.2M8.6 14.2h6.8' stroke-width='1'/><circle cx='12' cy='8' r='1.25' fill='currentColor' stroke='none'/><circle cx='8.6' cy='14.2' r='1.25' fill='currentColor' stroke='none'/><circle cx='15.4' cy='14.2' r='1.25' fill='currentColor' stroke='none'/>";
}
// (Build 7, #126: the Node child's own glyph — Build 6's #111
// server tower — retired with the rail child it was drawn for.
// Apex Node keeps its DHD page and its Servers row; it simply
// has no rail slot and therefore no rail glyph.)

function chipdeckGlyphSvg() {
  return "<circle cx='6.2' cy='5.4' r='1.9'/><circle cx='6.2' cy='12' r='1.9'/><circle cx='6.2' cy='18.6' r='1.9'/><path d='M9.4 3.6h9.1a1.9 1.9 0 0 1 1.9 1.9v13a1.9 1.9 0 0 1-1.9 1.9H9.4'/><path d='M15.9 8.2l3.1 4.6h-6.2z'/>";
}
function dossierGlyphSvg() {
  // The Dossier (v0.5.6e, #260): the person file — candidate
  // B, the mark Alex picked from three: a folder tab, the
  // head-and-shoulders on the record, two ruled lines.
  return "<path d='M3.5 6.5a2 2 0 0 1 2-2h4l2 2.5h7a2 2 0 0 1 2 2v8.5a2 2 0 0 1-2 2h-13a2 2 0 0 1-2-2z'/><circle cx='10' cy='11.2' r='1.9'/><path d='M6.9 17.3c.5-1.5 1.7-2.3 3.1-2.3s2.6.8 3.1 2.3'/><path d='M15.4 12h3.4M15.4 15.2h3.4'/>";
}
function navIcon(name) {
  if (name === "Network") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><rect x='2.5' y='3.5' width='19' height='7' rx='1.8'/><rect x='2.5' y='13.5' width='19' height='7' rx='1.8'/><circle cx='6.5' cy='7' r='1' fill='currentColor' stroke='none'/><circle cx='6.5' cy='17' r='1' fill='currentColor' stroke='none'/></svg>";
  if (name === "Utilities") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M14.7 6.3a4.5 4.5 0 0 0-6 6L3 18l3 3 5.7-5.7a4.5 4.5 0 0 0 6-6L14 13l-3-3 3.7-3.7z'/></svg>";
  if (name === "ChipDeck") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='1.8' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-deck'>" + chipdeckGlyphSvg() + "</svg>";
  if (name === "Mesh") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-mesh'>" + meshGlyphSvg() + "</svg>";
  if (name === "Federation") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-fed'><circle cx='12' cy='5' r='2.6'/><circle cx='5' cy='18' r='2.6'/><circle cx='19' cy='18' r='2.6'/><path d='M10.2 7.2L6.6 15.6M13.8 7.2l3.6 8.4M7.6 18h8.8'/></svg>";
  if (name === "FediChat") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-fedichat'>" + fedichatGlyphSvg() + "</svg>";
  if (name === "Dossier") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-dossier'>" + dossierGlyphSvg() + "</svg>";
  if (name === "Navi") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M12 2.5l2.3 7.2 7.2 2.3-7.2 2.3L12 21.5l-2.3-7.2-7.2-2.3 7.2-2.3z'/></svg>";
  if (name === "SIEM") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><circle cx='12' cy='12' r='8.6'/><circle cx='12' cy='12' r='4.6'/><path d='M12 9.4c.9 1.3 1.7 2.1 1.7 3.2a1.7 1.7 0 1 1-3.4 0c0-1.1.8-1.9 1.7-3.2z'/></svg>";
  if (name === "Casework") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><rect x='3' y='3' width='8' height='8' rx='1.5'/><rect x='13' y='3' width='8' height='8' rx='1.5'/><rect x='3' y='13' width='8' height='8' rx='1.5'/><rect x='13' y='13' width='8' height='8' rx='1.5'/></svg>";
  if (name === "DHD") return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' class='nav-ico-dhd'>" + dhdGlyphSvg() + "</svg>";
  return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'>" + (NAV_ICON_PATHS[name] || "") + "</svg>";
}
// Build 3 (#210): the app opens to Home \u2014 the fused Wire
// surface. Chat is one click down the rail, as it always was.
var view = "ThreatIntel";
var navClosed = false;
// Menu group state (Build 6, #114 — supersedes Build 4's
// expanded-by-default + persistence): every group starts
// COLLAPSED at every launch, on every screen size, no
// exceptions; toggling within a session works as it always
// has, and the group state is neither read from nor written
// to storage — a stale navi.navGroups save from an older
// build is cleaned on sight and never honored. (The rail's
// own open/collapsed state is NOT part of this ruling: it
// keeps its remembered behavior — see the navClosed read
// below. The correction is recorded in the build report.)
var navGroupsOpen = { systems: false, navi: false, apps: false, siem: false, utilities: false };
try { localStorage.removeItem("navi.navGroups"); } catch (e) {}
// The tabbed status pane (0.1.9e Build 6, issues #107/#108):
// Status is the default pane — the first render and every
// opening of the mobile dock land on it — and Chats is the
// second tab. Exactly one pane shows at a time.
var statusPaneTab = "status";
try {
  var savedNavClosed = localStorage.getItem("navi.navClosed");
  if (savedNavClosed !== null) navClosed = savedNavClosed === "1";
  // Small screens (build .61): with no saved preference, the menu
  // starts closed — the compact top bar (logo, tagline, one Menu
  // button) — instead of opening as the full destination stack.
  // A saved preference wins on every screen size, and desktops are
  // unaffected (the rail default there is open, as always).
  else if (window.matchMedia && window.matchMedia("(max-width: 800px)").matches) navClosed = true;
} catch (e) {}
// The status dock (build .64): on small screens the status column
// lives behind the dock toggle in the top bar, as a full-width
// overlay. Closed is the resting state on every screen size; it is
// never persisted — a page always comes up content-only.
var dockOpen = false;
// Status column visibility on desktop (Build 6, #114, as
// corrected): hidden at every launch, behind the pane button
// in her presence bar (Build 8; the menu chrome's text toggle
// was retired by the operator's ruling) — and deliberately
// NOT persisted: "closed by default" means every launch, so
// no storage key exists for it. On phones the dock (dockOpen
// above) is the column's other door, as always.
var statusHidden = true;
// navi.llamafile connection state, driven by the status element: it
// starts stopped ("Disconnected. Click to Connect."); only a successful
// connect probe flips it to running. While stopped, chat is a plain
// command line.
var llama = { status: "stopped", endpoint: "http://localhost:8080", error: null };
// Sudo mode (Build 3, #209): NO password lives in this page at
// all. Arming happens in Settings \u2192 System \u2014 a masked prompt
// whose value rides exactly one /api/sudo/arm request into the
// bridge's "sudo -v" and is forgotten by both sides. The page's
// only sudo truth is sudoState, read from /api/status; CONFIG.sudo
// mirrors its enabled flag so chat payloads can ask for elevation.
// Whether the Sudo Mode Warning dropdown is open — render() rebuilds
// the status column, so the open state lives here, not on the element.
var sudoWarnOpen = false;
// Local network access (build .62): whether its Access details
// dropdown is open, and which copy button last succeeded — both
// page-level for the same reason as sudoWarnOpen above.
var localNetOpen = false;
var dhdDetailsOpen = false;
// Servers group (build .79): the status column's Servers
// section is a collapsible parent over the three server rows.
// Expanded by default; the operator's choice is remembered
// across re-renders exactly like localNetOpen / dhdDetailsOpen.
var serversOpen = true;
var dhdUrlCopied = false;
var localNetCopied = "";
// The whole sudo warning, one line (operator's words).
var SUDO_WARNING_LINE = "Sudo Mode Gives Navi Admin Privs... Be mindful so you dont learn the find out part of the fuck around equation. <3";
// The chat composer's latched mode (its CLI / Navi press-once
// toggles): unmarked lines route this way until the other toggle
// is pressed or a line's $ / > prefix flips it. Page-level because
// render() rebuilds the composer on every state change. Persisted
// (build .67) in localStorage navi.chatMode: the latch is restored
// on load — toggle state, the bar's mode tint, and the placeholder
// all read chatMode at render — and every change (a toggle press
// or a line's $ / > prefix) writes it back, so the mode survives
// sends, view switches, and reopening the app. Navi is the
// default when nothing is saved.
var chatMode = "navi";
try { var savedChatMode = localStorage.getItem("navi.chatMode"); if (savedChatMode === "cli" || savedChatMode === "navi") chatMode = savedChatMode; } catch (e) {}
function persistChatMode() { try { localStorage.setItem("navi.chatMode", chatMode); } catch (e) {} }
// Navi Mesh (Ring 1, build .85): the per-chat local-only escape —
// when latched, this chat session never borrows a paired machine.
// The chat's route (v0.4.1e, #228): the composer's pill is
// the route control, and the BRIDGE is its one source of
// truth — this page holds only the bridge's last answer for
// the active chat (chatRoute), renders the face from it, and
// asks the bridge to flip. Face, stored state, and the view
// after a reload agree because all three read the same
// answer; a refusal lands in chatRouteMsg ON the pill row and
// the face does not move. (The pre-#228 "This chat only"
// latch kept its state in this page's localStorage and could
// disagree with the road the bridge actually took; that
// latch is retired, and the bridge still honors a meshLocal
// field from an older page caught mid-update.)
var chatRoute = null;        // the bridge's /api/chat/route answer for chatRouteChatId
var chatRouteChatId = null;  // the chat that answer belongs to
var chatRouteMsg = "";       // a flip refusal, in the bridge's words, shown on the pill row
function refreshChatRoute() {
  var cid = activeChatId;
  if (CONFIG.backend !== "live" || cid === null || cid === undefined) return;
  chatRouteChatId = cid;
  fetch("/api/chat/route?chatId=" + encodeURIComponent(cid)).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.ok && String(activeChatId) === String(cid)) { chatRoute = j; if (view === "Chat") render(); }
  }).catch(function () {});
}
function routeStops() {
  // The stops a flip can land on, in cycle order, computed
  // from the bridge's access table: Local always; Mesh when a
  // paired machine can be borrowed from; one stop per joined
  // federation with a borrowable member. A mode without
  // access is never a stop — it cannot be flipped into.
  var stops = [{ mode: "local" }];
  var avail = (chatRoute && chatRoute.available) || { mesh: false, fedinets: [] };
  if (avail.mesh) stops.push({ mode: "mesh" });
  (avail.fedinets || []).forEach(function (f) { stops.push({ mode: "federation", fedinetId: f.id, name: f.name }); });
  // v0.4.3e (#235): a chat carrying a stored route gains one
  // final stop — Automatic — which hands the chat back to the
  // precedence (the bridge clears the stored route; the face
  // that lands is the automatic face). A chat with nothing
  // stored is already automatic and has no such stop: there
  // is nothing to hand back.
  if (chatRoute && chatRoute.route) stops.push({ mode: "automatic" });
  return stops;
}
function routePillHtml() {
  if (!chatRoute) return "<button type='button' id='chat-route-pill' class='routepill' disabled aria-label='Loading this chat's route from the bridge'>Route…</button>";
  var eff = chatRoute.effective || { mode: "local" };
  var stored = chatRoute.route || null;
  var avail = chatRoute.available || { mesh: false, fedinets: [] };
  // v0.4.2e (#231): an effective route the bridge computed
  // from the automatic precedence (nothing stored) is marked
  // as what it is — an "Auto · " prefix and a dashed border —
  // so an automatic face can never be mistaken for a route
  // the operator chose, nor a chosen face for the default.
  // The bridge's answer carries the distinction ("automatic");
  // the face only renders it.
  var auto = !!chatRoute.automatic;
  var cls = "routepill", face = "", label = "";
  if (auto) cls += " routepill-auto";
  // v0.4.3e (#236): the faces are one word — Local / Mesh /
  // FediNet — so the pill fits the composer line on a phone
  // instead of taking a row of its own. The long forms live
  // in the descriptive label, which is also where a
  // federation's NAME now rides (the green face no longer
  // carries it): with several fedinets joined, the label is
  // what tells them apart. The automatic mark is unchanged:
  // an "Auto · " prefix and the dashed border.
  if (eff.mode === "mesh") {
    if (stored && stored.mode === "mesh" && !avail.mesh) {
      face = "Mesh Unavailable"; cls += " routepill-na";
      label = "This chat's route is the Mesh, but no paired machine can be borrowed from right now — replies answer locally until one can. Press to choose another route.";
    } else if (auto) {
      face = "Auto · Mesh"; cls += " routepill-mesh";
      label = "Automatic route: Mesh — no federation road is available, so the automatic precedence (federation, then Mesh, then local) puts this chat on the pairwise Mesh. Press to choose a route yourself.";
    } else {
      face = "Mesh"; cls += " routepill-mesh";
      label = "Route: Mesh — this chat's model work borrows over the pairwise Mesh. Press to change the route; the cycle's last stop, Automatic, hands the chat back to the automatic precedence.";
    }
  } else if (eff.mode === "federation") {
    var fed = null;
    (avail.fedinets || []).forEach(function (f) { if (f.id === eff.fedinetId) fed = f; });
    if (fed) {
      face = (auto ? "Auto · " : "") + "FediNet"; cls += " routepill-fed";
      label = auto
        ? "Automatic route: FediNet — " + fed.name + " — the automatic precedence (federation, then Mesh, then local) puts this chat on that federation while a member can be borrowed from. Press to choose a route yourself."
        : "Route: FediNet — " + fed.name + " — this chat's model work borrows through that federation. Press to change the route; the cycle's last stop, Automatic, hands the chat back to the automatic precedence.";
    } else {
      face = "FediNet Unavailable"; cls += " routepill-na";
      label = "This chat's federation route is not available right now — replies answer locally. Press to choose another route.";
    }
  } else if (auto) {
    face = "Auto · Local";
    label = "Automatic route: Local — no federation or Mesh road is available, so the automatic precedence (federation, then Mesh, then local) answers on this machine. Press to choose a route yourself.";
  } else {
    face = "Local";
    label = "Route: Local — this chat's model work runs on this machine. Press to change the route; the cycle's last stop, Automatic, hands the chat back to the automatic precedence.";
  }
  var h = "<button type='button' id='chat-route-pill' class='" + cls + "' aria-label='" + esc(label) + "' title='" + esc(label) + "'>" + esc(face) + "</button>";
  if (chatRouteMsg) h += "<span class='route-msg' role='status'>" + esc(chatRouteMsg) + "</span>";
  return h;
}
function cycleChatRoute() {
  if (!chatRoute) return;
  var stops = routeStops();
  var eff = chatRoute.effective || { mode: "local" };
  var idx = -1;
  for (var i = 0; i < stops.length; i++) {
    if (stops[i].mode === eff.mode && (eff.mode !== "federation" || stops[i].fedinetId === eff.fedinetId)) { idx = i; break; }
  }
  var next = stops[(idx + 1) % stops.length];
  if (!next) return;
  if (next.mode === eff.mode && next.fedinetId === eff.fedinetId) return;
  chatRouteMsg = "";
  fetch("/api/chat/route", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId: activeChatId, route: next }) }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.ok) {
      chatRoute = j; chatRouteChatId = activeChatId;
      // v0.4.3e (#235): the return to automatic is NAMED on
      // the pill row, the way a refusal is — the operator
      // sees the hand-back land, not just a face change.
      if (next.mode === "automatic") chatRouteMsg = "Handed back to Automatic — no route is stored for this chat; the precedence (federation, then Mesh, then local) routes it again.";
    }
    else chatRouteMsg = (j && j.error) || "The bridge refused that route — the route is unchanged.";
    render();
  }).catch(function () { chatRouteMsg = "The bridge did not answer — the route is unchanged."; render(); });
}
var meshBorrowNoticed = false;  // the first-borrow notice fires once per page session
var meshPrivacyAcked = false;   // the first-enable borrowing privacy copy, seen and saved past
try { meshPrivacyAcked = localStorage.getItem("navi.meshPrivacyAck") === "1"; } catch (e) {}
// Her voice: the last /api/voice answer (settings + engine
// availability), the live call as this page drives it (turns mirror
// the bridge's record), and the push-to-talk recorder. In mock mode
// the call is simulated with the browser's own speech synthesis and
// the mock chat brain — labeled as such, never passed off as the
// real engines.
var voiceState = null;       // live: last GET /api/voice answer
var micState = "unknown";    // unknown | ok | refused | unavailable — this window's own mic experience, shown on the Voice page
var hardwareState = null;    // live: last GET /api/hardware answer ({profile}); mock: baked seed profile
var hardwareMsg = "";
var publicState = null;      // live: last /api/status publicServer report; mock: the baked CONFIG shape
var dhdState = null;         // live: last /api/status dhd report; mock: the baked CONFIG shape (build .74)
var meshState = null;        // live: last /api/status mesh report (build .85); mock: derived from the baked CONFIG shape
var openCaseState = null;    // live: last /api/status openCase report (Build 15, #161) — the running case, read true
var meshMsg = "";
var meshTunnelArmed = {};    // Build 5 (#179): peers whose Backup route switch stands flipped on locally, awaiting the SSH host/user the bridge requires — UI state only, never posted until the fields stand filled
var meshPngMsg = "";         // the PNG card import's own verdict, shown at the drop zone (Build 7)
var meshTunnelConfirmFor = null; // Build 8: the peer whose tunnel-key crown confirm strip stands open on its card
var meshTunnelAuthMsg = null;    // Build 8: the tunnel-key act's verdict, shown on the peer's own card
var meshHealth = null;       // the last /api/mesh/health report (0.2.0e Build 1, #143); null until the operator runs the check
var meshHealthBusy = false;  // a health check is in flight
var meshHealthMsg = "";      // the health check's own fetch failure, when the bridge did not answer
var meshPngDrag = false;     // the drop zone is lit while a file drags over it (Build 7)
var meshDetailsOpen = false;
var dhdMsg = "";
var doorMsg = "";            // the Door (trusted devices) card's own message (Build 2, #173)
var dhdCodeShown = false;
var dhdOperation = null;     // the page's current DHD operation card state
var opPollTimers = {};       // operation id -> poll timer (chat + DHD page cards)
// Build .65: whether the freshest bridge answer actually carried a
// publicServer report. The Local network access section's ON face
// is a claim about the bridge answering RIGHT NOW, so it stands
// only while this is true — a last-known report from a bridge that
// has since stopped answering (it exited to apply the OFF choice,
// flipped from the phone itself) must not keep "On your network",
// the LAN address, or the access code alive on screen.
var publicReportLive = false;
var publicMsg = "";
var publicRestartPolling = false; // a toggle/restart confirmation poll is running (build .61)
// Confirmation polling for a Local server restart (build .61):
// after the toggle (or Restart bridge) saves, the bridge exits and
// the supervisor relaunches it — this page polls /api/status until
// the bridge that answers reports the saved choice live with the
// restart confirmed, or the window runs out. Running out is a
// NAMED failure (step: the new bridge answering), never a silent
// pending: the row says the bridge did not come back and offers
// Restart bridge and the quit-and-relaunch fallback.
function pollPublicRestart() {
  if (publicRestartPolling) return;
  publicRestartPolling = true;
  var tries = 0;
  var tick = function () {
    tries++;
    fetch("/api/status").then(function (r) { return r.json(); }).then(function (j) {
      publicReportLive = !!(j && j.publicServer);
      if (j && j.publicServer) publicState = j.publicServer;
      var ps = publicState;
      if (ps && !ps.pending && ps.restart && ps.restart.status === "confirmed") {
        publicRestartPolling = false;
        publicMsg = ps.restart.message || "Restart complete — the live state matches your choice.";
        render();
        return;
      }
      if (ps && ps.restart && ps.restart.status === "failed") {
        publicRestartPolling = false;
        render();
        return;
      }
      if (tries >= 25) {
        publicRestartPolling = false;
        if (publicState) {
          publicState.restart = { status: "failed", step: "answer", message: "The new bridge did not answer in time, so the saved choice could not be confirmed live. Still live: " + (publicState.activePublic ? "Local — on your network (the old bridge, untouched)." : "This machine only — 127.0.0.1:" + BRIDGE_PORT + " (the old bridge, untouched).") + " Press Restart bridge to try again, or quit Apex fully and relaunch." };
        }
        render();
        return;
      }
      setTimeout(tick, 1000);
    }).catch(function () {
      // The bridge is mid-restart (or gone): that is the expected
      // silence while it swaps. Keep polling inside the window.
      publicReportLive = false;
      if (tries >= 25) {
        publicRestartPolling = false;
        if (publicState) {
          publicState.restart = { status: "failed", step: "answer", message: "The bridge did not come back — nothing answers at this address, so no new state is live. Relaunch Apex; if Apex was started from a terminal that has since closed, nothing was left running to relaunch the bridge for it." };
        }
        render();
        return;
      }
      setTimeout(tick, 1000);
    });
  };
  setTimeout(tick, 800);
}
function fetchHardware(thenRender) {
  if (CONFIG.backend !== "live") { hardwareState = { profile: (CONFIG.hardwareProfile || null) }; if (thenRender) render(); return; }
  fetch("/api/hardware").then(function (r) { return r.json(); }).then(function (j) { hardwareState = j; if (thenRender) render(); }).catch(function () { hardwareState = { profile: null, error: "The bridge did not answer /api/hardware — a bridge from before build .34 has no hardware profile. Relaunch after reinstalling." }; if (thenRender) render(); });
}
var voiceWarmed = false;     // the bridge's engines were asked to warm this session
var voiceFetching = false;
var voiceMsg = "";
var call = null;             // {startedAt, turns:[{who,text,exec,ts}]} while a call runs
var badgeMsg = "";           // Operator Badge minting note (issue #116)
var badgeFaceChoice = "photo"; // badge face: the picture, or generated art
var callState = "idle";      // idle | listening | thinking | speaking
var callTimer = null;
var callSeconds = 0;
var callStarting = false;    // a /api/voice/call/start request is in flight
var mediaRecorder = null;
var mediaChunks = [];
var mediaStream = null;
var pttTarget = null;        // "call" | "chat" — where a recording goes
var speakQueue = [];
var speakPlaying = false;
// Bridge build handshake: bridge.py reports its build stamp from
// /api/status. This page carries the stamp of the package build it
// shipped in. A mismatch means the process answering on the bridge
// port is an older bridge that was never restarted after a reinstall
// — it serves this (new) dashboard from disk while running old code,
// which is exactly how chat 400s and sudo-auth 404s present. The
// status column says so, loudly, instead of leaving the operator to
// decode bare error strings.
var BRIDGE_BUILD = "2026-11-19.41";
var SHIPPED_BUILD_SEQ = 41;
var BRIDGE_PORT = 8675;
var bridgeStale = false;
var pageStale = false;   // build .91: the reverse mismatch — this PAGE is older than the bridge answering (an update landed under an open dashboard); the status column offers a reload
var bridgeBuildSeq = 0;  // build .91: the buildSeq the answering bridge last reported
var bridgeRoot = "";   // live: the bridge's install tree, from /api/status (build .86, issue #79)
// Known chats: every conversation is saved; the Chats tab switches
// between them. "messages" always references the active chat's list.
var chats = null;
var activeChatId = 1;
var messages = [];
var memory = [];
// Chat uploads (build .49): an attachment lives in the thread where
// it was uploaded. A text file's contents inline as conversation
// context with the next message; an image is stored (as a data URL,
// capped) and referenced by name — the local model is text-only, so
// nothing anywhere claims she can see it. Every upload is also
// indexed in navi.uploads so the Chats tab's "Uploaded files"
// section can find and delete it later.
var pendingUpload = null;
var uploads = [];
try { uploads = JSON.parse(localStorage.getItem("navi.uploads") || "[]"); } catch (e) { uploads = []; }
function persistUploads() { try { localStorage.setItem("navi.uploads", JSON.stringify(uploads)); } catch (e) {} chatsWriteThrough(); }
var docs = [];
// Updater state (build .47; end-to-end apply in build .56): the
// real check's last answer plus the apply's live record, kept in
// one place so the Updater view and the status column's UPDATER
// element read the same truth. state: idle (never checked) |
// checking | uptodate | available | failed (the check could not
// complete — error says why) | applying (the one-click apply is
// running; step names where it is: downloading | verifying |
// staging | installing | restarting) | updated (the apply
// finished and the relaunched app's VERSION read confirmed it;
// fromLabel → toLabel are the actual versions) | applyfailed
// (a step failed; applyError names the step and the reason, and
// stagedDir + command carry the manual fallback when staging
// was reached). Nothing here is simulated: in Mock backend
// mode no check runs at all (checks go through the live bridge),
// and the view says so.
var updater = { state: "idle", channel: (typeof CONFIG !== "undefined" && CONFIG.updaterChannel) || "stable", latest: "", releaseName: "", releaseUrl: "", publishedAt: "", prerelease: false, buildSeq: null, currentBuildSeq: SHIPPED_BUILD_SEQ, bgTag: "", assetName: "", assetSize: 0, notes: "", notesSource: "", currentRelease: null, error: "", stageError: "", stagedDir: "", command: "", step: "", fromLabel: "", toLabel: "", applyError: "", tailFailure: "", pollTimer: null, diagnoseCommand: "" };
// Whether the channel on screen is the operator's own pick
// (Build 2): the bridge stamps a panel choice as explicit and
// reports both the effective channel and the stamp on
// /api/status and every check. An Edge build sitting on Stable
// WITHOUT the stamp is on its inherited default — which the
// bridge resolves to Edge — never a mismatch; WITH the stamp it
// is a real choice, and the panel says what that choice costs.
var updaterChannelExplicit = false;
// Chips screen: three tabs — Navi User Repo (NUR) (the GitHub navi-chips
// catalog + installs, served live by the bridge), Chip Studio (the