---
name: agenthub
description: Drive the agenthub CLI to give an AI client MCP tools - add a server (curated catalog, pasted config, or by hand), store its credentials, run its OAuth login, test it for real, connect it to Claude Code / Cursor / Codex / VS Code / Zed, narrow what each client may see, and record the JSON-RPC wire when one misbehaves. Use whenever the task involves adding, authorizing, testing, or wiring up an MCP server, or diagnosing one a client cannot reach or that answers wrongly.
---

# agenthub

One gateway between every AI client and every MCP server. A client is
configured **once**, against agenthub; servers are added, authorized and
narrowed here afterwards without touching the client again.

Everything below was verified against the **released** binary. If a command
you want does not appear here, run `agenthub <group> --help` — **do not guess
a flag.** A plausible-looking command that does not exist is worse than no
command, because it fails after the user has pasted it somewhere that matters.

**This file stays on the surface a release build advertises** — `server`,
`auth`, `secret`, `catalog`, `profile`, `client` — and teaches nothing else.
Other command groups do exist and do run, but a release deliberately keeps
them off `--help` so the binary recommends one route; a command the user
cannot find in their own `--help` is one they cannot check you on. (`agenthub
connect` is advertised as well, under "machine entry point": it is the command
a client's own config runs, never one to type — see §5.)

## Before anything else

```bash
agenthub server ls           # what is already configured
agenthub client ls           # which clients are wired up, and what each may see
```

That is the whole orientation, and it is enough: the first tells you what is
already configured, the second which clients are already wired to it (add
`agenthub client detect` when you need the config paths themselves).
Nothing else needs to run before you know what the user is asking for.

### Always use `--json`, and branch on the exit code

`--json` is a global flag on every command. The envelope is stable:

```json
{"ok":true,"data":[],"warnings":[]}
{"ok":false,"error":{"code":"E_SERVER_NOT_FOUND","message":"no server \"nope\"","hint":"run 'agenthub server ls' to see configured servers"}}
```

Exit codes are frozen by design. **Branch on these, never on prose** — the
wording of a message may change, these may not:

| code | meaning |
|---|---|
| 0 | success |
| 1 | generic (downstream, network, internal) |
| 2 | usage error — you got the flags wrong, re-read `--help` |
| 3 | not found (server / profile / secret / skill) |
| 4 | the daemon is offline and this command needs it |
| 5 | authentication failure → the server needs `agenthub auth login` |
| 6 | refused by governance (HITL deny, quarantine) — **not** a bug to route around |
| 7 | a lock is held, or stored state is corrupt and cannot self-heal |

Exit 6 deserves care: something deliberately said no. Report it and stop;
disabling the control that produced it is not a fix.

Exit 7 is **not registry-only**, and reading it that way sends you to the
wrong file: any of agenthub's cross-process locks produces it — the registry,
the integrity baselines, the skills store, the token store — as does a state
file that is corrupt beyond self-healing. So "which store" is a question the
message answers and the exit code does not.

## 1. Add a server

**`server add` writes the definition and nothing else — the entry lands
DISABLED.** `server enable <id>` is the second step, and it is where the
connection probe runs. Adding is configuration; enabling is putting it into
service, and only the second one needs the network.

Two exceptions enable for you: **`catalog add`** composes add + enable, and
**`auth login`** enables the server it just authorized — that is what keeps
the OAuth path two commands (`add` → `login`) rather than three.

`enable` reports what the probe found and **enables either way**: a server
needing a login is enabled and says so. Pass `--no-probe` to skip the dial.

### a. From the server's URL or command — the normal case

Most MCP servers are a URL. Add it directly:

```bash
agenthub server add remote --url https://mcp.example.com/mcp
agenthub server add local-dev --url http://127.0.0.1:3000/mcp --local

# stdio (a process on this machine)
agenthub server add my-server --cmd npx --args "-y,@scope/mcp-server"

# then, in every case:
agenthub server enable <id>
```

`--local` is required for a literal loopback URL and allows **only** that; it
never opens up RFC1918 addresses. Other useful flags: `--env KEY=VALUE`,
`--header KEY=VALUE` (both repeatable), `--cwd`, `--transport`, and the
`--oauth-*` pins in §3.

If the server needs authorization, add it first and then §3 — `server add`
does not need the credential up front, and the `auth login` there enables it,
so no separate `server enable` is needed on that path.

### b. From a config the user already has

If they paste a `mcpServers` block, a `claude mcp add-json` line, or another
client's config, feed it in whole instead of translating it by hand:

```bash
echo "$JSON" | agenthub server add my-server --stdin
agenthub server enable my-server
```

Unknown keys are **rejected**, not silently dropped. An error here means the
paste really does contain something agenthub does not model — read it, do not
strip fields to make it pass.

To adopt what a client already has, read its config and paste the entries
back in — there is no bulk import:

```bash
agenthub client inspect cursor          # every location, and what is in each
```

### c. From the curated catalog — a shortcut, not the main road

The catalog holds a small hand-maintained set. It is worth a look when the
user names a well-known server and you would otherwise be guessing at its
package name or parameters; for anything else, go back to (a).

```bash
agenthub catalog search github          # or: agenthub catalog ls
agenthub catalog show filesystem        # description, target, params, the exact add line
agenthub catalog add fetch              # entries that need nothing: added AND enabled
agenthub catalog add filesystem --param directory=/Users/me/projects
agenthub catalog add github --name gh   # --name when the default id collides
```

`catalog show` prints the add command with its parameters filled as
`<placeholders>`. Read it rather than composing your own. A miss here means
nothing is wrong — most servers are simply not in it.

### Container isolation

```bash
agenthub server add sketchy --runtime docker --image ghcr.io/x/server:tag \
  --mount /Users/me/data:/data:ro --network none --memory 512m
```

`--runtime docker` is honoured or the command **fails** — it never quietly
degrades to running on the host. Default network is `none`; mounts are
read-only unless you write `:rw`.

That promise is checkable rather than taken on trust: `agenthub server inspect
<id>` prints `spawns`, the exact `docker run` argv the spawner would execute.
Read it when a container's isolation matters — the mounts, the network and the
limits that actually reach `docker` are on that one line.

## 2. Credentials — only for servers that take an API key

Skip this section unless the server authenticates with a key you have in
hand. OAuth servers are §3, and they store their own credentials; a server
that needs nothing needs nothing here.

`secret` sits beside `auth` in Setup because the two answer the same question
for different servers: `auth` for the ones that hand out their own credential,
`secret` for the ones that take a key the user already holds.

The registry is plain configuration and must never hold a credential, so the
key needs somewhere else to go. Put a **reference** in the definition and the
value in the vault:

```bash
agenthub server add brave --cmd npx --args "-y,@modelcontextprotocol/server-brave-search" \
  --env "BRAVE_API_KEY=\${SECRET_BRAVE_API_KEY}"

printf %s "$TOKEN" | agenthub secret set brave BRAVE_API_KEY --stdin
agenthub server enable brave        # after the secret exists, so the probe can succeed
```

**Never put a credential in an argv.** It lands in shell history and in every
`ps` on the machine. `secret set` reads no-echo from the terminal, or from
stdin with `--stdin` for pipelines.

The placeholder and the stored name must match: `${SECRET_<KEY>}` in the
definition, `secret set <server> <KEY>` in the vault. Resolution happens at
**dial time** and is fail-closed — an unresolved placeholder refuses the
connection rather than sending the literal `${SECRET_X}` upstream, which would
come back as a 401 indistinguishable from an expired token. HTTP servers take
the same form: `--header "Authorization=Bearer \${SECRET_X}"`.

`agenthub server inspect <id>` prints the env and headers back, so **which
placeholder landed where** is readable — that is the point of showing them. The
one exception is a literal `Authorization` value: a literal there is a pasted
token, and inspect will not read it out to a terminal.

The rest of the group:

```bash
agenthub secret ls [server]                 # KEY NAMES and backends, never values
agenthub secret rm brave BRAVE_API_KEY      # from every writable backend
agenthub secret migrate --from keyring --to enc-file [server] --dry-run
```

**There is no read path.** Values cannot be printed back, by design. If a user
asks you to show a stored secret, the honest answer is that nothing can. There
is no `secret get`, and asking for its help now says so: `agenthub secret get
--help` and `agenthub help secret get` both exit **2** with `unknown command`.
(Up to 0.11.0 they exited 0 printing the group's page, which reads exactly like
a real command's answer — if you are on an old build, do not take that page as
evidence that `get` exists.)

Two things `secret ls` tells you that are easy to misread:

- **`BACKEND` is the level a lookup would actually hit**, not where you put it. Resolution is four levels, first hit wins: `AGENTHUB_SECRET_<KEY>` in the environment, then a bare `<KEY>` (only with `AGENTHUB_ALLOW_BARE_SECRET_ENV=1`), then the encrypted file, then the OS keyring. So a row reading `env` means an environment variable is shadowing what you stored — that is the answer when a key you just set appears not to take effect.
- **A listed key is set, but an empty or whitespace-only value counts as unset** at every level, so a server can still fail to authorize with the key sitting right there in the listing.

`--scope` defaults to `_global` and is only for derived instances of one
server holding their own credentials. Leave it alone unless the user raises
it. `migrate` moves values only between `keyring` and `enc-file`; environment
variables are per-process input, not a backend, so nothing migrates into or
out of them.

## 3. OAuth login

**You cannot complete this for the user.** It needs a browser and a human.
Your job is to run the command that produces the URL and hand it over.

```bash
agenthub auth status                    # who is authorized, never any credential
agenthub auth login github              # auto-selects a flow
agenthub auth login github --manual     # prints the URL, reads the pasted callback
agenthub auth login github --device     # RFC 8628, for a machine with no browser
agenthub auth refresh github
agenthub auth logout github
```

In a non-interactive session prefer `--manual` or `--device`: `--loopback`
wants to open a browser on this machine and will simply time out (180s
default) where there is none.

When discovery fails, pin what the provider does not advertise —
`--issuer`, `--scopes`, `--redirect-uri`, `--authorization-endpoint` — or
store the pins on the entry with `server add --oauth-issuer` /
`--oauth-scope` / `--oauth-resource-metadata` so every later login uses them.

A successful `auth login` also **enables** the server, so an OAuth downstream
is `server add` → `auth login` and nothing more.

Exit 5 anywhere else means the credential expired: `auth login` again.

## 4. Test — before wiring anything to a client

This is the step that separates "the definition is stored" from "it works".

```bash
agenthub server test brave                       # connect, list tool names
agenthub server test brave --tools               # with compact signatures
agenthub server test brave --schema search       # one tool's full input schema
agenthub server test brave --tool search --args '{"query":"hello"}'   # a REAL call
```

`--timeout` defaults to 120s because a cold `npx`/`uvx` cache genuinely is
that slow — a first run that seems to hang usually is not hung.

`--tool` really invokes the tool. Treat it as you would any other side effect:
read-only tools freely, anything that writes only when the user asked for it.

## 5. Connect the client

```bash
agenthub client ls                           # who is wired up, and on which profile
agenthub client detect                       # ids, config paths, writability
agenthub client connect claude-code --dry-run    # ALWAYS look first
agenthub client connect claude-code
```

`--dry-run` prints the exact JSON that would be merged **and the file it would
land in**. Show both to the user before writing — this edits a file they own.

**Connect writes the user-level file by default** (`~/.claude.json`,
`~/.cursor/mcp.json`, …), because the entry carries this machine's absolute
agenthub path and a project-level file is meant to be committed. Pass
`--placement project` only when the user asks to wire up one tree — and say
that the path inside it is machine-specific. To narrow what a client sees, use
a profile (§6), never the file location.

The written entry runs `agenthub connect --client <id>`, so **servers added
later need no further client changes**. That is the whole point of the
gateway; do not go back and edit client configs per server.

`client detect` reports `writable`, and it is decided by the config's **format,
not its contents**: the two JSON shapes are written, and TOML, YAML and the
fileless remote shape are not — that is `codex`, `continue` and `open-webui`,
and nothing else on the list.

**Comments are not a reason any more.** A `settings.json` carrying the vendor's
comment header — Zed ships one, VS Code's file is JSONC by convention — is read
normally, and written by splicing agenthub's own entry into the existing bytes:
comments, key order and indentation survive because nothing else is rewritten.
The splice is proved before it reaches the disk, so a locator that cannot place
the entry costs you a refusal plus the manual snippet, never a mangled config.

When a client is **not** writable, the real `client connect <id>` is what gets
you the snippet — **not `--dry-run`**. `--dry-run` always renders the JSON
`mcpServers` entry, the wrong syntax to paste into a TOML or YAML file, while
the real command writes nothing, exits 1 (`E_CLIENT_UNSUPPORTED`) and prints the
fragment in that client's own format; under `--json` it is the `hint`. `codex`
is the exception, immediately below: its connect delegates to the client's own
CLI and really does succeed. `open-webui` has no file at all — it consumes MCP
over HTTP, so what comes back is an endpoint to register on the Open WebUI side.

`client detect` prints two lists under the table, and the difference between
them is the answer here. `supported clients:` is **every** client agenthub knows
about — it is not the writable subset, and it is deliberately not filtered,
because it is what answers "why is my client missing". The line below it,
`agenthub does not write these itself:`, names the three, with the command that
says what to do instead. (Releases up to 0.11.0 printed one list labelled
`directly writable clients:`, which named `codex` on the same screen where
codex's own row said `no`. If you see that label, you are on an old build and
the `WRITABLE` column is the answer.)

For **codex** that is not a dead end: `client connect codex` runs
`codex mcp add` for them, after backing the file up and before verifying the
result by reading it back. It needs `codex` on PATH; without it the command
refuses (**exit 1, `E_GENERAL`**, never a silent no-op) and prints what to
run. `client disconnect codex` runs `codex mcp remove` the same way, and
names the entry that is actually in the file rather than assuming it is
called `agenthub`.

Pass `--manual` when the user does not want agenthub executing another
program; `AGENTHUB_NO_CLIENT_CLI=1` does it for the whole machine. Either way
they get the instructions instead.

For **continue** (YAML) there is no such CLI: agenthub neither reads nor
writes it, so `client ls` says `?` and the entry goes in by hand.

Never hand-edit a format to get around a refusal without saying so — the
refusal is about protecting the file, not about agenthub's convenience.

**The client must be restarted to pick up the change.** Tell the user — an
unrestarted client looks exactly like a broken gateway.

`agenthub client ls` reports whether the entry is actually in each client's
file. Read its CONNECTED column literally: `yes` / `no` are answers, and
`denied`, `unreadable` and `?` are not — they mean agenthub could not read
the file, could not parse it, or does not parse that format at all, and none
of the three is fixed by running connect again. `agenthub client inspect <id>`
says which file and why. `--stat-only` skips the reads entirely (no macOS
privacy prompt) at the cost of answering `?` for everything, and `--all` lists
every supported client, installed here or not.

`unreadable` is now a stronger statement than it used to be: the file does not
parse **even with its comments blanked out**. So it means real syntax damage —
a truncated write, a stray bracket — and the fix is in the file, not in
agenthub. Comments alone no longer produce it.

For codex specifically, `?` means the file is there but written in a way
agenthub's TOML reader does not model — not that the entry is missing.

`agenthub client disconnect <id>` reverses it. With no target named it clears
the user-level file and, only if nothing of ours is there, the project-level
one — so an entry written by an older agenthub is still removed.

## 6. Narrow what each client sees

Default is every enabled server. Narrow with a profile, then bind clients:

```bash
agenthub profile create research
agenthub profile server add research brave                # <profile> then <server>
agenthub profile tools research brave --only search       # or --all / --none
agenthub profile discovery research lazy                  # or grouped / full / -
agenthub client bind cursor research
agenthub client ls                                        # connected? and on which profile
agenthub client unbind cursor                             # back to the fallback profile
agenthub profile use research                             # the fallback every UNBOUND client gets ("-" clears it)
```

`profile tools` takes the server's **own** tool names (`search`), not the
`brave__search` the client displays; `agenthub server test <id> --tools`
lists them.

A misspelled name is **stored anyway and warned about** — the rule has to be
writable before anything has ever connected, when there is no recorded tool
list to check it against. Read those warnings: `--only` is an intersection, so
one misspelling lets **nothing** through for that server, and the command still
says OK. And no warning is not a verification — a server whose catalog has
never been fetched has nothing to check against, which is `no opinion`, not
`no problem`. `agenthub server test <id> --tools` first is what makes the
check real.

All narrowing lives on the **profile**; a client only selects one. There is no
`agenthub scope` group, and a client binding never carries servers or tools of
its own. Two clients that need different surfaces get two profiles.

A binding takes effect on **sessions that are already running** — the gateway
recomputes and pushes `tools/list_changed`. Rebinding needs no client restart
(unlike `client connect`, which edits the client's own file).

Binding to a profile that does not exist is accepted, warns, and fail-closes
that client to an **empty** scope. If a client suddenly sees nothing, check
`agenthub client ls` for a `MISSING` marker before suspecting the servers.

The layers **intersect** for security fields: a narrower layer can only take
capability away, never add it. If a tool is unexpectedly missing, look for a
narrowing layer before suspecting the server.

### Discovery: how the surface is presented

`profile discovery` changes how tools are *surfaced*, never which tools are in
the set. Pick by size, not by preference:

| mode | what `tools/list` returns | use when |
|---|---|---|
| `full` | every visible tool, one entry each | small surfaces, or a client that filters for itself — ask for it explicitly |
| `grouped` | one aggregate entry per server, plus `call_tool` last | a mid-sized set: the client reads per-server entries first, then dispatches |
| `lazy` | the meta-tools (`search_tools` / `describe_tool` / `call_tool` / …) plus any pinned tools | large surfaces — the client's context holds a handful of names instead of hundreds. **This is the default when nothing sets a mode** |
| `-` | clears the profile's override | fall back to the global default |

So a client nobody configured gets `lazy`, and finds tools by calling
`search_tools` rather than by reading a list it was handed. On a small surface
that trade is not worth it — put the client on a profile and say
`agenthub profile discovery <profile> full` (§6), rather than assuming the
tool is missing.

Visibility is decided by the **scope**, never by the mode: `lazy` hides names
from the initial list, it does not take capability away. Narrowing is §6's
job. An unrecognised value degrades to the default rather than erroring, which
is why `profile discovery` rejects a typo at write time.

The one kill switch that applies everywhere, above every profile:

```bash
agenthub server disable brave       # whole server, every client, at once
```

No profile can put a disabled server back, and it needs nothing to have
connected first — it is the right instrument when a server has never come up
at all, or when the answer is "not this server, not anywhere".

**"Remove it" almost always means `disable`, not `rm`.** `server rm` deletes
the whole footprint and does not ask twice: the stored credential, profile
membership, governance rules naming it, integrity baselines, approval grants
and the cached tool list all go with the entry. Secrets cannot be read back,
so an OAuth login or an API key destroyed here is destroyed — the user
re-authorizes from scratch. Audit records survive.

So: `disable` when they want it to stop, `rm` only when they say they want it
gone permanently — and say what `rm` takes before running it, because the
command itself will not.

To take away **one tool** rather than a whole server, that is profile work:
`profile tools <profile> <server> --only …` on the profiles the affected
clients are on, plus `profile use` for the fallback that unbound clients get.
Two clients that must differ get two profiles; there is no global per-tool
switch on this path.

## 7. Verify end to end

```bash
agenthub server inspect brave --tools   # what is recorded for one server, offline
agenthub server test brave --tools      # what it answers today, live
agenthub client ls                      # who is wired up, and what each may see
```

`inspect` reads the cache — the last contact, not a handshake made now — so it
answers "what would a cold gateway serve". `test` connects and answers "what
does this server say today". When the two disagree, `test` is the truth and
the cache is stale.

**None of that proves a client is using any of it.** A written config file
shows intent; the confirmation is the client itself, restarted, calling a tool
and getting an answer back. When you need to see that the call arrived and
what the server replied, record it — §8.

Exposed names are `<server>__<tool>`, but **never split on `__` to recover
them** — a server id or a tool name may itself contain it. Take the server
from the listing you asked for and the tool name from `server test --tools`.

### `server inspect` also answers "who can see this server"

`inspect` prints a **visibility** section, and it is the one place the two
halves of that question are joined. Reach for it on the failure that otherwise
has no obvious next command: the server is enabled, the credential is stored,
and the client still shows nothing.

```bash
agenthub server inspect brave        # no --tools needed; visibility is always printed
```

It names the profiles that **include** the server, each with its tool selector;
the profiles that **exclude** it (which a list of the others cannot tell you);
the bound clients that can and cannot reach it; and — always, not only when it
changes the answer — what a client with **no binding of its own** gets, because
that is exactly what the person reading is unsure of. Three states stay
distinct because they need different repairs: a **disabled** server reaches
nobody whatever the profiles say, an excluded profile is one `profile server
add` away, and a binding pointing at a profile that no longer exists
fail-closes to an **empty** scope — which from outside is indistinguishable
from deliberate exclusion. The count of local tool overrides rides along, worth
knowing before you compare this report against the names a client displays.

Two properties make it the right first command rather than a nicer one:

- **It is computed from the registry alone.** No client config file is opened — that is `client inspect`'s deliberate, per-client act, and it can raise a macOS privacy prompt — and no daemon is needed, so the answer stays available on exactly the machine that is broken.
- **It is an upper bound, not a live claim.** The scope chain only ever narrows, so a session can hold less than this. It reports what the configured bindings allow, never what a running session got.

## 8. When something does not work

| symptom | look here |
|---|---|
| exit 3 | check the id: `server ls`, `catalog ls`, `profile ls` |
| exit 4 | the command needed a running daemon. Nothing on the path above does — re-read what you actually ran |
| exit 5 | `agenthub auth status`, then `auth login <server>` |
| exit 6 | governance refused, deliberately. Report it and stop; do not route around it |
| exit 7 | some store's lock is held by another process, or its state is corrupt. The message names which one. Retry once, then say so — do not delete state to clear it |
| server will not connect | `server test <id>` first — it prints the real error and the child's stderr tail |
| a tool vanished | `server inspect <id>` — its visibility section names the profile that narrowed it (§6); then a quarantine, before suspecting the server |
| a server you added is invisible | `add` leaves it DISABLED — `agenthub server enable <id>` |
| client sees nothing | `server inspect <id>` first: which profiles reach it, and what an unbound client gets (§7). Then — did the user restart it? `client detect` to confirm the entry is in the file |
| it connects, but a tool answers wrongly | record the wire — `server trace <id> on`, reproduce, `server logs <id>` (below) |

### Recording the wire

Once a server passes `server test` and still misbehaves inside the client,
the question is no longer "is it reachable" but "what exactly did it say".
Record the traffic and read it back:

```bash
agenthub server trace linear on      # off by default, per server
# the user reproduces the problem in their client
agenthub server logs linear          # -f/--follow to watch it live
agenthub server trace linear off     # once you have the answer
```

**`server logs` only reads; `server trace` is what fills the file.** An empty
log means nothing was recorded — **not** that the server sat idle. It shows
the last 100 frames by default; `--limit 0` is all of them, and `--json`
carries the full payloads (the human table truncates its DETAIL column).

Three properties decide how to use it:

- **It takes effect immediately.** A client that is already running starts recording without being restarted, and the server being investigated is not reconnected — you are reading the same connection that is misbehaving.
- **Frames are captured before redaction.** Whatever the server actually returned sits verbatim in `<data>/logs/server-<id>.log`, the path `server trace` prints. That is the point of it, and the reason to turn it off again. Read it; do not paste one wholesale into a chat, an issue or a commit.
- **It is per server, and it persists.** Nothing expires a trace, so one left on keeps recording across restarts. `server ls` grows a `TRACE` column while anything is traced, and only then — that is where to look when nobody remembers which server was left recording.

There is no per-client or per-profile trace, and looking for one is wasted
time: every session reaching a server shares one connection, so narrower
recording cannot be honoured and is refused rather than approximated.

A trace is the wire, not the gateway's own reasoning: it shows what the server
said, never why agenthub decided something. If the frames look correct and the
client still misbehaves, the problem is above this layer — check §6 for a
narrowing that removed the tool.

## Rules that are not style preferences

- **Never write a credential into an argv or into the registry.** `secret set --stdin`, and `${SECRET_X}` in the definition.
- **Never invent a command or a flag.** `--help` is authoritative; this file may lag the binary.
- **Never treat exit 6 as an obstacle.** Something refused on purpose. Report it.
- **Show `--dry-run` output before editing a user's client config.**
- **A trace is temporary and it is raw.** `server trace <id> off` when you have the answer, and never quote a trace file wholesale — it holds downstream responses captured before anything redacts them.
- **`server test` before `client connect`.** Wiring an unverified server means the user debugs it inside their client, where the error is least legible.
- **Say when a step needs a human** — the OAuth browser flow and the client restart both do. Claiming an unverifiable success is worse than reporting the handoff.
