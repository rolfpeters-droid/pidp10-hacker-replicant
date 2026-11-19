# PiDP-10 Hacker Replicant

An AI tutor/copilot that sits next to Oscar Vermeulen's PiDP-10 replica: a
web page with the real operator console on one side, and a chat panel on
the other that knows **ITS**, **TOPS-10**, and **TOPS-20** (the historic
PDP-10 operating systems this replica can boot) — and, if you let it, can
type into the console itself.

This is part of a small family of "Hacker Replicants" built for different
PDP/VAX/HP replicas. The pattern (a web console + a chat panel + an
LLM backend that can optionally read/type into the console) is shared; this
repo is PiDP-10-specific in its chat UI theme and its tutor's knowledge.

**Important difference from the sibling repos:** this one does **not**
bundle a standalone boot-API script. See "Wiring up machine control" below
for why, and what to do instead.

## How it works

```
Browser
 │
 ├─ Left panel:  <iframe> → your existing PiDP-10 web console (CONSOLE_URL)
 │
 └─ Right panel: chat → POST /api/chat (SSE stream)
      │  this Node service (server.js)
      │
      ├─ LLM backend: any OpenAI-compatible /chat/completions API
      │  (local Ollama by default, or a hosted provider -- see below)
      │
      └─ when the control toggle is ON, two tools only:
           send_keys   → POST  <BOOTAPI_BASE>/keys    {"text": "...", "line": "terminal"|"console"}
           read_screen → GET   <BOOTAPI_BASE>/screen?line=terminal|console
           (served by two routes added to your own PiDP-10 web console --
            see "Wiring up machine control")
```

The PiDP-10 is unusual among these replicas in exposing **two separate
serial lines**, not one: a user terminal line (where you log into TOPS-10
or TOPS-20) and a separate operator console line (front-end/DDT and the OS
console). Both tools take an optional `line` parameter so the agent can
pick the right one — see `skills/pdp10-tutor.md` for when each matters, and
the ITS caveat (ITS doesn't use either serial line for its login TTYs; it
drives a simulated display instead, so the agent's reach there is more
limited than under TOPS-10/TOPS-20).

## Wiring up machine control

Every Hacker Replicant needs exactly two primitives to let the chat agent
operate the machine: **read the current screen content**, and **type text
into the live session**. The sibling PiDP-11/PiDP-8i repos ship a tiny
standalone script for this (`screen -X stuff` / `screen -X hardcopy`
against a GNU `screen` session) because that's how those machines' console
sessions are implemented.

The PiDP-10 this was built against doesn't use `screen` for its console —
its existing web console already holds an open **telnet** connection to the
simulator, with a small in-memory buffer per line. That state lives inside
the FastAPI process of the web console itself, which made it impractical to
split out into a separate tiny script the way the `screen`-based machines
could: you'd either need to duplicate the telnet connection (risky — now
two processes fighting over the same line) or have the script talk to the
web console's internals anyway.

So instead, the two Hacker Replicant routes were added **directly to the
existing web console app** as two extra FastAPI endpoints, reusing its
already-open telnet connection and line buffers. `docs/pidp10-console-routes.py.example`
is a worked, adapted-to-be-generic version of exactly that: a recipe for
implementing `GET /screen`, `POST /keys`, and (optionally) `GET /status`
against a FastAPI app that already tracks an open line somewhere. Read it
as "here's how I did it for a telnet-backed console", not as a drop-in
script.

What this means for `BOOTAPI_BASE` in your `.env`: point it at wherever
you've actually added these two routes.

- **If your PiDP-10 setup is like the one this was built against**
  (a FastAPI web console with an open telnet line to the simulator): add
  the routes from `docs/pidp10-console-routes.py.example` directly to that
  app, adapted to your own internal names for the line buffer/writer, and
  point `BOOTAPI_BASE` at that same app (e.g. `http://localhost:8080/api/pidp10`
  if it's the same process serving `CONSOLE_URL`).
- **If your PiDP-10 setup drives its console through GNU `screen` instead**
  (more common on other PDP replicas): you don't need any of the telnet
  plumbing above. Use the sibling PiDP-11/PiDP-8i `bootapi.py` as your
  template instead — the `screen -X stuff <text>` / `screen -X hardcopy
  <file>` pattern ports over directly and is considerably simpler than the
  telnet version.

Either way, the chat backend (`server.js`) doesn't care how `BOOTAPI_BASE`
is implemented — it just expects `GET /screen`, `POST /keys`, and
(optionally, for the status note in chat) `GET /status` under that base
path, in the same shape used by the other tools in this family.

## LLM-agnostic by design

The chat backend talks to **any backend that speaks the OpenAI-compatible
`/chat/completions` API with tool/function calling** — that's one code
path for both:

- **Local** (default, free): point it at a local [Ollama](https://ollama.com)
  install. Ollama has spoken this API since mid-2024, nothing extra to
  configure beyond pulling a model that supports tool calling (`qwen3`,
  `llama3.1`+, `mistral-nemo`, etc. all do).
- **Cloud** (optional, for a faster/smarter option): point it at OpenAI,
  Groq, Together, Fireworks, DeepSeek, Mistral, OpenRouter, a hosted
  vLLM/TGI endpoint — anything OpenAI-compatible. See `.env.example` for
  examples.

No vendor SDK, no hard dependency on any single provider, no API key
required at all if you only use the local backend.

**A heads-up on local "thinking" models:** Ollama's OpenAI-compatible
endpoint (what the local backend uses) doesn't currently expose a way to
turn off a reasoning model's internal "thinking" step the way Ollama's own
native API does (`think: false`) — tested directly against Ollama 0.x as of
Nov 2026, passing `think: false` in the request body had no effect via
`/v1/chat/completions`. For `qwen3` specifically this means every reply
carries a few hundred tokens of hidden reasoning, adding roughly 10-20
extra seconds per turn compared to what you'd get calling Ollama's native
API directly. `LLM_TIMEOUT_MS` already accounts for this (default 180s), so
nothing breaks, it just feels slower than it has to. If snappier replies
matter more to you than using a reasoning model locally, point
`LOCAL_MODEL` at a non-thinking model instead (`llama3.1`, `mistral-nemo`,
etc. all support tool calling without this overhead).

## Setup

### Prerequisites

- A PiDP-10 running Oscar Vermeulen's SIMH-based software, with its own web
  console already working (whatever that looks like in your setup — a
  FastAPI/telnet bridge like the one this was built against, a `screen`
  session, or something else).
- Two routes added somewhere reachable over HTTP, per "Wiring up machine
  control" above, if you want the control toggle to work. The chat panel
  is fully usable as a knowledge tutor without this — the toggle just won't
  do anything useful until `BOOTAPI_BASE` points at something real.
- Node.js 18+ on the host running the chat backend.
- [Ollama](https://ollama.com) running somewhere reachable (for the free
  local backend), or an API key for a cloud provider.

### Install

```bash
git clone <this-repo-url>
cd pidp10-hacker-replicant
npm install
cp .env.example .env
# edit .env: at minimum set CONSOLE_URL, and BOOTAPI_BASE if you've wired up control
```

### Run the chat backend

```bash
npm start
```

Open `http://<host>:8099/` (or whatever `PORT` you set) — `CONSOLE_URL`
should point at your existing PiDP-10 web console.

### Running it as a service

See `docs/pidp10-hacker-replicant.service.example` for a ready-to-adapt
systemd unit for the chat backend. There's no separate boot-API service
example here — see "Wiring up machine control" above for why there's
nothing standalone to run.

## Configuration reference

All configuration is environment variables — see `.env.example` for the
full list with examples. The short version:

| Variable | What |
|---|---|
| `PORT` | Port the chat backend listens on |
| `CONSOLE_URL` | Your existing PiDP-10 web console, shown in the left panel |
| `BOOTAPI_BASE` | Where the two Hacker Replicant routes are reachable (see "Wiring up machine control") |
| `ACTION_LIMIT` | Tool calls per turn before the agent must ask to continue |
| `LOCAL_BASE_URL` / `LOCAL_API_KEY` / `LOCAL_MODEL` | Local (default) backend |
| `CLOUD_BASE_URL` / `CLOUD_API_KEY` / `CLOUD_MODEL` | Optional cloud backend |

## Safety model

- The control toggle is **off** on every page load — by default the agent
  only talks, it never touches the machine.
- When on, the agent gets exactly two tools (`send_keys`, `read_screen`),
  nothing else — no shell, no filesystem, no network access beyond the two
  routes behind `BOOTAPI_BASE`.
- After `ACTION_LIMIT` tool calls in one turn, the agent is cut off from
  further tool calls and must ask in plain text whether it may continue —
  no silent, unbounded action streak. A hard iteration cap (20) sits
  underneath as a defensive backstop against a broken loop.
- Everything the agent types is visible live in the same console session
  your own web console shows — no black box.
- This does **not** protect you from the agent typing something wrong (it's
  an LLM, it can make mistakes) — the boundary is on *what* it can touch,
  not on *how well* it types. Use a local/free backend if you want to
  experiment without any cost risk.

## Repository layout

```
server.js                        Chat backend (Node/Express, SSE, LLM-agnostic)
public/                           Chat panel UI (right side)
skills/pdp10-tutor.md             System prompt: ITS/TOPS-10/TOPS-20 knowledge + behavior rules
docs/
  pidp10-hacker-replicant.service.example   Example systemd unit for the chat backend
  pidp10-console-routes.py.example          Worked example/recipe for the two machine-control routes
```

There's no `webconsole/` or bundled `bootapi.py` in this repo — see "Wiring
up machine control" above.

## Adapting this for a different replica

`skills/pdp10-tutor.md` is the only file that's actually PDP-10-specific
content; swap it for a system prompt about your own machine and point
`server.js` at the new file. The two tool definitions (`send_keys`,
`read_screen`) and the SSE/action-limit plumbing in `server.js` are
reusable as-is for any machine that exposes the same two routes — only
`docs/pidp10-console-routes.py.example` (or, if you're using `screen`, the
sibling repos' `bootapi.py`) needs to change to match how your machine's
console is actually implemented.

## Credits

Built by Rolf Peters with Seb (his homelab AI assistant), in collaboration
with Oscar Vermeulen (designer of the PiDP replica series). Inspired by
Oscar's own `agent-pdp1` project for the PiDP-1.

## License

MIT — see `LICENSE`.
