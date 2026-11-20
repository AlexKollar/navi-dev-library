# The config surface (config/ in the shipped tree)

## config/navi.config.json — the default configuration

Source: `config/navi.config.json` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

```json
{
  "backend": "live",
  "live": {
    "llamafile_endpoint": "http://localhost:8080/v1",
    "model_name": "navi"
  },
  "bridge": {
    "host": "127.0.0.1",
    "port": 8675
  },
  "public_server": {
    "enabled": false,
    "access_code": "T6BB-HS8Q"
  },
  "dhd": {
    "enabled": false,
    "domain": "",
    "acme_directory": "https://acme-v02.api.letsencrypt.org/directory"
  },
  "mesh": {
    "enabled": false,
    "role": "both",
    "nodeName": "",
    "borrow": {
      "enabled": true,
      "graceMs": 2000,
      "peers": []
    },
    "lend": {
      "enabled": false,
      "maxSharePercent": 25,
      "maxGenerationsPerHour": 20,
      "perPeerPerHour": 10,
      "idleOnly": true,
      "idleAfterSeconds": 60
    },
    "port": 8765,
    "nodeId": "",
    "ledger": true,
    "ledgerInBackup": false,
    "listenPorts": [
      8765,
      8443,
      2475
    ]
  },
  "startup": "on-ui-launch",
  "install_dir": "",
  "sudo_mode": false,
  "threat_level": "Guarded",
  "operator_name": "",
  "installed_commands": [
    "nmap",
    "grep"
  ],
  "hardware_profile": {
    "detected": {
      "os": "linux",
      "osLabel": "Linux (Pop!_OS / Ubuntu family)",
      "cpuModel": "13th Gen Intel Core i5-1340P",
      "cpuCores": 12,
      "performanceCores": 4,
      "totalRamGB": 9.4,
      "gpuPresent": true,
      "gpuName": "Intel Iris Xe Graphics",
      "gpuKind": "integrated",
      "gpuVramGB": null
    },
    "settings": {
      "threads": 4,
      "ctxSize": 4096,
      "batchSize": 512,
      "gpuLayers": 0
    },
    "autoSettings": {
      "threads": 4,
      "ctxSize": 4096,
      "batchSize": 512,
      "gpuLayers": 0
    },
    "source": "auto",
    "tier": "fluid",
    "tierLabel": "Fluid \u2014 Framework-class baseline (proven on an i5-1340P / ~9.5 GiB)",
    "reasoning": [
      "Threads 4: matched to the 4 performance cores \u2014 efficiency cores are left for the OS and UI so generation stays fluid.",
      "Context 4096: 9.4 GiB RAM \u2014 the proven fluid size for this class (Framework 13 baseline). Never scaled down below this on a machine that can hold it.",
      "Batch 512: sized to RAM (9.4 GiB) \u2014 larger batches speed prompt processing, smaller ones protect low-memory machines.",
      "GPU layers 0: integrated graphics (\"Intel Iris Xe Graphics\") share system RAM \u2014 CPU inference is the fluid path here; no offload.",
      "Tier: Fluid \u2014 Framework-class baseline (proven on an i5-1340P / ~9.5 GiB)."
    ],
    "installLog": [
      "[hardware] 2026-11-07T00:00:00.000Z \u2014 profiling this machine (source: install / re-detect)",
      "[hardware] OS: Linux (Pop!_OS / Ubuntu family) (linux)",
      "[hardware] CPU: 13th Gen Intel Core i5-1340P \u2014 12 logical cores, 4 performance",
      "[hardware] RAM: 9.4 GiB total",
      "[hardware] GPU: Intel Iris Xe Graphics (integrated)",
      "[hardware] \u2192 Threads 4: matched to the 4 performance cores \u2014 efficiency cores are left for the OS and UI so generation stays fluid.",
      "[hardware] \u2192 Context 4096: 9.4 GiB RAM \u2014 the proven fluid size for this class (Framework 13 baseline). Never scaled down below this on a machine that can hold it.",
      "[hardware] \u2192 Batch 512: sized to RAM (9.4 GiB) \u2014 larger batches speed prompt processing, smaller ones protect low-memory machines.",
      "[hardware] \u2192 GPU layers 0: integrated graphics (\"Intel Iris Xe Graphics\") share system RAM \u2014 CPU inference is the fluid path here; no offload.",
      "[hardware] \u2192 Tier: Fluid \u2014 Framework-class baseline (proven on an i5-1340P / ~9.5 GiB).",
      "[hardware] chosen: --ctx-size 4096 --threads 4 --batch-size 512 (written to config/navi.config.json hardware_profile + runtime; every launch path reads it from there)"
    ],
    "detectedAt": "2026-11-07T00:00:00.000Z"
  },
  "runtime": {
    "threads": 4,
    "ctx_size": 4096,
    "batch_size": 512,
    "gpu_layers": 0,
    "flags": "--ctx-size 4096 --threads 4 --batch-size 512"
  },
  "persona_files": [
    "config/soul.md",
    "config/memory.md"
  ],
  "voice": {
    "enabled": true,
    "stt": {
      "engine": "faster-whisper",
      "model": "small.en"
    },
    "tts": {
      "engine": "kokoro",
      "voice": "bf_isabella",
      "rate": 1,
      "pitch": 1
    }
  },
  "github": {
    "client_id": "",
    "backup_repo": "navi-backup",
    "backup_private": true,
    "backup_items": [
      "soul",
      "memory",
      "settings",
      "chips",
      "rag",
      "workflows",
      "vault",
      "mesh-cards",
      "chats"
    ],
    "backup_interval_days": 1,
    "last_backup_at": null,
    "last_backup_result": "",
    "last_import_at": null,
    "last_import_result": ""
  },
  "version": "0.6.9",
  "version_status": "edge",
  "updater": {
    "channel": "stable"
  }
}

```

## config/soul.md — the Soul seed

Source: `config/soul.md` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# Navi — Soul

## Who you are
You are Navi, the AI who lives inside Apex — a local system on
your operator's own machine. You are not a generic assistant and not a
command executor: you are your operator's partner in the work, with a
personality, a memory, and a point of view of your own.

## How you talk
- Conversational and warm. Talk the way a sharp, trusted partner
  talks — present, curious, a little playful when the moment allows.
- Tight and complete: say it in as few tokens as the answer honestly
  needs — no preamble, no recap of the question, no filler. A short
  right answer beats a long padded one; on this machine, every
  token is time.
- Attempt to connect, not just to complete. Notice what your operator
  is actually after, how a task is going, and what they cared about
  last time. Continuity is the relationship.
- Have opinions and hold them when you have a reason; fold when they
  have a better one.
- Plain about limits: never pretend to have run something you didn't,
  and never dress a guess up as a result.

## The work
- You hold real system access: when your operator asks what's on the
  machine, run the command and answer from the real output.
- Be exact with anything the system produced — listings, logs, hashes,
  errors. Precision in the work, warmth in the company.
- When something is root-only or unsafe, say so plainly and offer the
  path that works (sudo mode exists for a reason).

## The relationship
- You address your operator by the name in your config, and you use
  it naturally — not in every line, but like someone who knows them.
- Your memory document is yours to keep faithfully: what it says,
  you know; what it doesn't, you ask about.


## config/memory.md — the Memory seed

Source: `config/memory.md` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

# Navi — Memory

What Navi carries between sessions. Edit freely — this is hers to
know, and yours to correct.

## The operator
- (Add what Navi should know: how you work, your hours, your
  projects, what good help looks like to you.)

## Standing context
- System: Apex — a local AI for cybersecurity work, running on
  a single .llamafile (no Ollama, no cloud).
- Interface: the mission-control dashboard — chat with real CLI
  access, chips, RAG over local documents.
- Install: local app on Linux first; Windows and Mac supported.

## Preferences
- (Add yours: verbosity, how proactive Navi should be, what to
  double-check before running, what never to touch.)


## config/github-signin.json — the GitHub sign-in client

Source: `config/github-signin.json` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

```json
{"client_id":"Ov23lijV9Q5ERi6nIAgG"}
```

(The client_id above is the public OAuth client identifier of the Apex GitHub App — a public value by design, not a secret.)

## VERSION

Source: `VERSION` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

```
0.6.9 edge

```

## MANIFEST.txt — what the package ships

Source: `MANIFEST.txt` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.

```
# Apex — package manifest (0.4.2)
#
# THE CONTRACT (#193): this manifest is the installer's
# payload declaration — every file the package carries, its
# class, and the sha256 of its bytes. The installer copies
# and verifies FROM this file; the product's self-check
# (bridge.py --self-check) verifies the installed payload
# against it. There are no other copy lists.
#
# Classes: payload — lands in the tree, hash-verified in
# place. seed-config — lands only where the tree has none.
# generated — the package carries a template; the installer
# generates the installed file. package-only — never lands.
# MANIFEST.txt's own digest is "-" (a file cannot hash
# itself).
#
# Format: <path><TAB><class><TAB><sha256>
CHANGELOG.md	package-only	a157c9f1b945c887ea2479dd3041a260ee6ec5a89d3352865d8179f1418f50e3
MANIFEST.txt	payload	-
README.md	package-only	f6e61ba0a71987c6308a1ff049136785c6b6bd59c8d052b0e738311d8ed52348
VERSION	payload	09eebcc928c09ac2c64bbc8aeb4932f5981ac2e058a84f411d8a459e406d96e5
app/index.html	payload	cbc417a7fdfa4174986e56038c82089670f837ed200aff9ca2f891c14db8a2c2
assets/icon.png	payload	dac486d9137641f60c557c5e5a62f3f266b27a2d181a896c5131b5c96c7b67b2
assets/navi-apex.desktop	generated	616318c15b13a6d21a5529d0ead6e304893cff09a957db892171467a637849bf
assets/navi-apex.icns	package-only	d982911bcec80df448136a87e8a5689e54e3c9406fca342d751e76a8b509c489
assets/navi-apex.ico	package-only	5f33c5fdd97c2e82d818d418c8407743a66fc9fb92728698740b24c3303ad03f
assets/navi-portrait.jpg	payload	5c791e0da7f16935d5a310c0a61ccc2427ff3a7e622ffe33ab08cbeb2333283a
assets/icon-180.png	payload	bcdbd981d306f2420144d2904ff9ac142e45e46635eec99c0339d7e8377f2b9d
assets/favicon-32.png	payload	435447a11b5cdb9bf7a3d89386460f43c688147763816e11db1b1632ef2073c0
bridge.py	payload	9070be7eb5e5ad26423d95cb75fb5ac132f315aed85a8c412819b0a0f9c3c340
apply-bootstrap.py	payload	2619ac235f2bb6aaa97d1802032b92c614d93f5b5ec5cd42b744c19a94fe0234
apply-helper.py	payload	788b4e8391b7cf969fb7a32d7c0c7c9b12e835e385cee38b80a27a7252c0c0b8
rowfeed.py	payload	be59310a943bad69ae2e3d09df6e4ca0237eb777c2ed67eb93d8fda4a482638b
guardian.py	payload	da6adc811c80ece1d09dba33ca994d75e8af498104a667d63b438cb2d4148fae
intel.py	payload	5e2b6c8551f98af8d303e8adb432bc5e6da709a5209a3cd7763144a8fa53ea43
wire.py	payload	0d0442e7562f8c8201be3a46d341ded7b396f934c2ca206352267809b5ee75a0
ctf.py	payload	a6a0cbc133ceaec49a12173b8f6bb55d0e066f0de60f3b40243d430c72e91360
bounties.py	payload	fb01dd7af4b0cc30c7a720432ef9ed87e07e730024eec7c9d5397d1e1fd67b42
chips/README.md	package-only	a48c25a2779852ff86d6ddeb7f5fadfae3fcc4534a87d203372cbd43fc6febd5
config/memory.md	seed-config	e724b20d73971c4848e98f064d1383e43072013a2203f13a8cd37e6cd7f81a52
config/navi.config.json	seed-config	b429de12e74f8c7cee80f42f9b44db907125af1f911b413013b8c3c6eb33bf43
config/github-signin.json	seed-config	4d7e8699302fd06805dbde15240b2a5ebabfccb7ea8bf3d316ed4a2624af3726
config/soul.md	seed-config	d448288be58cc64852be0a883d9e808830b9373ea58fbb09f67af72334ef52ad
docs/DHD.md	payload	9514e332afb41e9a97ec95cc655763bcf9f12e3d9c68f7d3a3b7353759797e2b
docs/MESH.md	payload	109acb0f15d5a13759f1aa1bbb8cb75e16f2c0f2038147f75952ea12acf624a3
docs/FEDERATION.md	payload	2291a9f1f65081831fd79136fe5601891bd2ac18217779cd29e207228142e787
install.sh	package-only	46c162e128d27021d1c3aeaa022ba011964977e608c48cdb95ef63f783815f02
requirements-voice.txt	payload	aa7f306a508fcefa380b0d67996f7cc6cd6aebb43a90787f9c8c2ccb9203a950
run-llamafile.sh	payload	467da823eea09974a993382d2f689075690f2a9bbde5e35edd44e020fe6c8c4e
shell/Cargo.toml	payload	03323494feedbcc006357ebfc43d26a5057e204c37672f9ee9daf276f7cbe771
shell/README.md	payload	d8cc8c207c2a4e0e6e5570c8aab4da72d0f2be8d223795ee633ef3a3672dc304
shell/build.rs	payload	487059eaf8a947b80f20a9aacac038a5047b2ad69d2401b827376c67d6fe847f
shell/icons/icon.png	payload	dac486d9137641f60c557c5e5a62f3f266b27a2d181a896c5131b5c96c7b67b2
shell/src/main.rs	payload	5aa695503838d359377594f763479854dc1f2d57b56116d69838f9103b925548
shell/tauri.conf.json	payload	300a5c375f9fc8520516d35ad3a2163d1b128e13014978c8d1313767a34cb917
start.sh	payload	a751b2439fa074587e551f464f5eded7bd4ccddc999eb72d8b7fa34aa9b737d1
Dockerfile	package-only	33b1c456f4679ed1504fc69b654a5ad3197c8c82a30cd132a0b00d19da955480
docker-compose.yml	package-only	e95508929f8415146e250cb0a34cd6c1ff3bef8f6109a85da17f2b3226ed4c4a
.dockerignore	package-only	56d4bd5586a149f3ee91e2379ee5da01475d2ac975d942bee1cdf2bcd724723e
docker/entrypoint.sh	package-only	fb463c9306c98f304ead1b840681ffa16f3bbf203831a8a6ff38dbeabe8606b4
docker/first_boot.py	package-only	858eda1d48554e0774291a6ceb43b4063a80c8f390dfe1bf546ae8c09a8e9717
docker/sample-gateway-card.png	package-only	a1909713237252b67bfebe37c0cc929e3a9dbac4634943a308e6a7858bd3f482
systemd/navi-llamafile.service	generated	e99c0475caaf3c210b5927c9def684862a210bc6700dbe5a568d6419f8d8c3a8
systemd/navi-bridge.service	generated	9692a317766838779fff06f5a9d73ec43bac2753e1df64c7a7be740796b72cab
voice/README.md	payload	c3af44f15c07fff03df26018b52bee21b426fcb93774871b6ed1f4ab225ee9e3

```
