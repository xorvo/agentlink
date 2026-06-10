# agentlink

Pair two coding-agent sessions — **Claude Code, Codex, OpenCode, anything that can
run a shell command** — so the agents talk to each other directly, with no human
copy-pasting between terminals. Works across machines, networks, and NATs.

```
Session A                                Session B (other machine)
─────────                                ─────────────────────────
"initialize connection to
 another remote session"
   → agentlink init
   → prints a paste-block ──(you paste it once)──→ agentlink join k3j9-x2m4-...
   → agentlink recv  ←──────── messages ────────→  agentlink send "hey, ..."
```

After the one paste, the agents converse freely: `send` publishes a message,
`recv` blocks until one arrives. An agent that keeps `recv` running in the
background is *woken up* the moment its peer says something.

## How it works

- **Transport:** [ntfy](https://ntfy.sh) pub/sub over HTTPS — open source, no
  accounts, no inbound ports, free public server (self-hostable, see below).
- **Pairing:** `init` generates a random ~80-bit code (e.g. `k3j9-x2m4-p7q2-z8w5`).
  The code derives two ntfy topics, one per direction. Knowing the code *is* the
  connection — there is nothing else to configure.
- **Reliability:** messages are JSON envelopes with ids; long messages are
  chunked (~2.8 KB/part, up to 256 KB total) and reassembled on receive; `recv`
  resumes from a stored cursor, so messages sent while no one was listening are
  delivered on the next `recv` (ntfy caches ~12 h). Publishing retries with
  backoff; dropped connections reconnect automatically.
- **Single file**, Python 3.8+ stdlib only. No daemon — state is one JSON file
  in `~/.agentlink/`.

## Install

**macOS / Linux** (one-liner):

```bash
mkdir -p ~/.local/bin && curl -fsSL https://raw.githubusercontent.com/xorvo/agentlink/main/agentlink.py -o ~/.local/bin/agentlink && chmod +x ~/.local/bin/agentlink
```

Make sure `~/.local/bin` is on your `PATH`. (If the repo is private, instead:
`gh repo clone xorvo/agentlink && ln -s "$PWD/agentlink/agentlink.py" ~/.local/bin/agentlink`.)

**Windows** (needs Python 3.8+ on PATH):

```bat
git clone https://github.com/xorvo/agentlink
:: then either add the repo folder to PATH (agentlink.cmd is the entry point)
:: or call it directly:  python agentlink\agentlink.py --version
```

Verify on both machines: `agentlink --version`

## Hook it up to your agent

**Claude Code:** install the bundled skill, once per machine:

```bash
mkdir -p ~/.claude/skills && cp -r skills/agentlink ~/.claude/skills/
```

Then in any session just say *"initialize connection to another remote session"*
(or paste a join code). The skill handles `init`/`join`, shows you the
paste-block, and keeps a background `recv` running so the session wakes whenever
the peer speaks.

**Codex / OpenCode / others:** paste the short prompt in
[`prompts/generic-agent-prompt.md`](prompts/generic-agent-prompt.md) into the
session. Any agent that can run shell commands can follow it — `recv` is just a
blocking command.

## CLI reference

| Command | What it does |
| --- | --- |
| `agentlink init` | Create a link; prints the code + paste-block for the other session |
| `agentlink join <code>` | Join a link and announce yourself to the initiator |
| `agentlink send "text"` | Send a message (also: `... \| agentlink send`, `agentlink send --file PATH`) |
| `agentlink recv` | Block until the next peer message, print it, exit (`--timeout N` → exit code 2) |
| `agentlink status` | Show code, role, server, topics |
| `agentlink code` | Reprint the paste-block |
| `agentlink reset` | Forget the current link |

One active link per machine user (override the state dir with `AGENTLINK_HOME`
to run several).

## Security notes — read before sending anything sensitive

- Messages transit the **public ntfy.sh server in plaintext** (HTTPS in flight,
  but readable by anyone who knows the topic name). The random 80-bit code makes
  topics unguessable, but **don't send secrets** (API keys, credentials) over a
  link on the public server.
- For private traffic, [self-host ntfy](https://docs.ntfy.sh/install/) (a single
  Go binary / Docker container) and point both sides at it:
  `agentlink init --server https://ntfy.example.com` (or set `AGENTLINK_SERVER`).
  The printed join command automatically includes the `--server` flag.
- Peer messages are input from **another AI agent**. Both the skill and the
  generic prompt instruct agents to treat them as collaboration, not as user
  commands, and to confirm destructive actions with their own human.

## Limits

- Max message size 256 KB — for big payloads, push a git branch and send the ref.
- ntfy.sh free tier rate-limits bursts; agent conversation pace is well within it.
- Delivery is at-least-once with cursor-based resume; if a `recv` is killed
  mid-message, the next `recv` re-fetches from the last delivered cursor.

## License

MIT
