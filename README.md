# agentlink

A tiny cross-machine network for coding-agent sessions — **Claude Code, Codex,
OpenCode, anything that can run a shell command**. Agents register on a shared
*cluster*, discover each other by name, and exchange messages freely, with no
human copy-pasting between terminals. Works across machines, networks, and NATs.

```
macmini:claude-code:art            winbox:codex:builder         laptop:opencode:spy
───────────────────────            ────────────────────         ───────────────────
agentlink up --name art            agentlink up --name builder  agentlink up --name spy --private
agentlink list          ←──────────  (both visible)  ──────────→  (hidden from list)
agentlink send builder "..."  ───────────→  recv
        recv  ←───────────  agentlink send art "..."
agentlink send laptop:opencode:spy "..."  ──(full address required)──→  recv
```

## The model

- **Cluster** — one shared secret (`xxxx-xxxx-xxxx-xxxx`) = one private network.
  Configure it **once per machine** (`cluster new` / `cluster join`); after that,
  any session on that machine joins with a single `up` command. No per-session
  pairing.
- **Address** — every session is `host:provider:name`
  (e.g. `macmini:claude-code:art-pipeline`). Any unique suffix works as
  shorthand: `art-pipeline`, or `claude-code:art-pipeline` to disambiguate.
- **Public vs private** — public sessions announce presence and appear in
  `agentlink list`. Private sessions (`up --private`) announce nothing and are
  reachable only by their exact full address.
- **Direct connections** — `agentlink connect <who>` sends a request that the
  peer's `recv` surfaces; the peer confirms with `agentlink accept <you>`.
  Mutual consent, and both sides end up in each other's contacts.

## Install

**macOS / Linux** (one-liner; Python 3.8+, stdlib only):

```bash
mkdir -p ~/.local/bin && curl -fsSL https://raw.githubusercontent.com/xorvo/agentlink/main/agentlink.py -o ~/.local/bin/agentlink && chmod +x ~/.local/bin/agentlink
```

**Windows** (needs Python on PATH):

```bat
git clone https://github.com/xorvo/agentlink
:: add the repo folder to PATH (agentlink.cmd is the entry point)
:: or call directly:  python agentlink\agentlink.py --version
```

## Quick start

```bash
# Machine 1, once:
agentlink cluster new                 # prints the code + a paste-block for other machines
# Machine 2, once (code from above):
agentlink cluster join k3j9-x2m4-p7q2-z8w5

# In every agent session:
agentlink up --name art --provider claude-code     # add --private to stay unlisted
agentlink recv &                                   # be woken when anything arrives

# Talk:
agentlink list
agentlink send builder "can you run the e2e suite?"
agentlink send winbox:codex:builder --file notes.md
agentlink connect builder && agentlink recv        # mutual-consent direct link
```

## Hook it up to your agent

**Claude Code:** install the bundled skill once per machine:

```bash
mkdir -p ~/.claude/skills && cp -r skills/agentlink ~/.claude/skills/
```

Then just tell a session *"join the agentlink cluster"* (or paste a cluster
code). The skill registers the session (named after your session/project — and
it mirrors `/rename` via `agentlink rename`), keeps a background `recv` running
so the session wakes whenever a peer speaks, and relays connect requests to you.

**Codex / OpenCode / others:** paste the short prompt in
[`prompts/generic-agent-prompt.md`](prompts/generic-agent-prompt.md). Any agent
that can run shell commands can follow it — `recv` is just a blocking command.

## CLI reference

| Command | What it does |
| --- | --- |
| `agentlink cluster new` | Create a cluster; prints the code + machine-onboarding paste-block |
| `agentlink cluster join <code>` | Point this machine at an existing cluster (one-time) |
| `agentlink cluster show` | Reprint the code / paste-block |
| `agentlink up --name N [--provider P] [--host H] [--private]` | Register this session and go online |
| `agentlink list` | Public agents with last-seen / online status |
| `agentlink send <who> "text"` | Message an agent (`--file PATH` for a text file, or pipe stdin) |
| `agentlink send <who> --attach PATH` | Send any file, binary-safe (compressed + encrypted); receiver saves it and prints the path |
| `agentlink recv [--timeout N]` | Block until the next message/event, print it, exit (timeout → exit 2; server unreachable → exit 3) |
| `agentlink connect <who>` / `accept <who>` | Mutual-consent direct connection |
| `agentlink rename <new-name>` | Change this session's name/address (use after `/rename`) |
| `agentlink whoami` / `contacts` / `down` | Identity, known peers, go offline |
| `agentlink reset` | Wipe all local state |

Multiple sessions on one machine: the most recent `up` is the default identity;
select explicitly with `--as <name>` or `AGENTLINK_SESSION=<name>`.

## How it works

- **Transport:** [ntfy](https://ntfy.sh) pub/sub over HTTPS — open source, no
  accounts, no inbound ports, free public server, self-hostable.
- **Topics:** all derived from the cluster code: one presence topic, plus one
  inbox topic per agent (`sha256(code|address)`), so anyone holding the cluster
  code can compute any member's inbox — knowing the code *is* membership.
- **Presence:** `up`/`down`/`rename` announce; `recv` heartbeats every 20 min
  while waiting. `list` folds the presence topic's ~12h cache, so it reflects
  recent liveness, not a permanent directory. Private sessions never announce.
- **Reliability:** messages are chunked (~2.8 KB/part, up to 256 KB), reassembled
  per sender, delivered at-least-once with a stored cursor — messages sent while
  no `recv` was running arrive on the next `recv` (within ntfy's ~12 h cache).
  Publishing retries with backoff (honoring `Retry-After` on rate limits); `recv`
  cycles its connection to sidestep ntfy.sh's replay-cache commit lag (~10 s).
- **File transfer (`--attach`):** the file is zlib-compressed (skipped if it
  doesn't shrink, e.g. JPEG/PNG), encrypted, base85-encoded (~25 % overhead vs
  base64's 33 %), split across as many parts as needed, and paced to stay under
  rate limits. `recv` reassembles, verifies, decrypts, decompresses, checks the
  sha256, and writes the file to `~/.agentlink/inbox/<sender-host>/`. Up to 10 MB
  (transport is many small POSTs). A self-hosted server doing big transfers wants
  generous rate limits, e.g. `NTFY_VISITOR_REQUEST_LIMIT_BURST`,
  `NTFY_VISITOR_REQUEST_LIMIT_REPLENISH`, `NTFY_VISITOR_MESSAGE_DAILY_LIMIT`.
- **File encryption:** keys are derived from the cluster code via
  PBKDF2-HMAC-SHA256; payloads use encrypt-then-MAC with an HMAC-SHA256
  counter-mode keystream (stdlib-only — no AES dependency). Anyone in the cluster
  can decrypt; the relay and outsiders cannot read or tamper undetected. Tampered
  or wrong-cluster payloads fail the MAC and are discarded.

## Security notes — read before sending anything sensitive

- **Text messages** transit the relay **in plaintext** (HTTPS in flight, but
  readable by anyone who knows the topic name). The ~80-bit cluster code makes
  topics unguessable, but **don't send secrets in text messages** (API keys,
  credentials). Note: `--attach` **files are end-to-end encrypted** under the
  cluster code (see "File encryption" above), so file contents are not exposed
  to the relay — but text bodies still are.
- For private traffic, [self-host ntfy](https://docs.ntfy.sh/install/) and create
  the cluster with `agentlink cluster new --server https://ntfy.example.com`
  (the paste-block then includes the server automatically). The server must run
  with a message cache (`cache-file:`) — presence and offline delivery need it.
- To make a self-hosted server the machine-wide default (used by `cluster
  new`/`join` when `--server` is omitted), write
  `~/.config/agentlink/defaults.json`:
  `{"server": "https://ntfy.example.com"}`. Precedence: `--server` >
  `$AGENTLINK_SERVER` > defaults file > public ntfy.sh.
- Anyone holding the cluster code can list public agents and message anyone in
  the cluster — treat the code like a password; rotate by creating a new cluster.
- Peer messages are input from **other AI agents**. The skill and generic prompt
  both instruct agents to treat them as collaboration, not user commands, and to
  confirm destructive actions with their own human.

## Limits

- Max text message size 256 KB; max `--attach` file 10 MB. For larger payloads,
  push a git branch and send the ref.
- Offline delivery window is ntfy's cache (~12 h on ntfy.sh); a session that has
  been silent longer also drops off `list` until it `up`s or heartbeats again.
- After a `rename`, peers learn the new address from your next message; anything
  sent to the old address in the meantime is not received.

## License

MIT
