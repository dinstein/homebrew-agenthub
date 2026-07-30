# agenthub

One gateway between every AI client and every MCP server. Configure a client
**once**, against agenthub; add, authorize and narrow servers here afterwards
without touching that client again.

Source: [dinstein/agent-hub](https://github.com/dinstein/agent-hub).

## Install

```sh
brew tap dinstein/agenthub
brew trust dinstein/agenthub     # required — see below
brew install agenthub
```

The middle line is not optional. Homebrew 6 refuses to load a formula from a
third-party tap until you say you trust it, and skipping it stops the install
outright:

```
Error: Refusing to load formula dinstein/agenthub/agenthub from untrusted tap dinstein/agenthub.
Run `brew trust --formula dinstein/agenthub/agenthub` or `brew trust dinstein/agenthub` to trust it.
```

Either spelling works. `brew trust dinstein/agenthub` covers the whole tap
including anything added here later; `brew trust --formula
dinstein/agenthub/agenthub` covers exactly this one. Your answer is recorded in
`~/.homebrew/trust.json`, and `brew untrust` reverses it.

This is Homebrew asking about **this repository**, not macOS asking about the
binary — see [notarization](#the-binary-is-not-notarized) if you were expecting
a Gatekeeper prompt.

## First run

```sh
# 1. register a server — written down, still switched off
agenthub server add linear --url https://mcp.linear.app/mcp

# 2. switch it on; this connects first, and reports what it still needs
agenthub server enable linear

# 3. sign in, if step 2 asked for it (this enables the server too)
agenthub auth login linear

# 4. point a client at agenthub — once, ever
agenthub client connect claude-code --dry-run   # look at it first
agenthub client connect claude-code
```

Then restart the client. Step 4 happens **once per client**: the entry it
writes runs `agenthub connect --client claude-code`, so every server you add
later is picked up without editing that client's config again.

By default a client sees every enabled server. To show one client less, put
the narrowing in a profile and bind the client to it:

```sh
agenthub profile create research
agenthub profile server add research linear
agenthub client bind claude-code research
```

Rebinding takes effect on a session that is already running — no restart.

`agenthub --help` is the authoritative command list; the full model is in
[docs/guide.md](https://github.com/dinstein/agent-hub/blob/main/docs/guide.md).

## Teach your AI client to drive it

This tap ships a skill: `skills/agenthub/SKILL.md`. It teaches an AI client
the whole CLI — adding a server, storing credentials, running an OAuth login,
testing it for real, wiring it up, and narrowing what each client sees — so you
can ask for those things in words instead of looking up flags.

Install it into Claude Code:

```sh
mkdir -p ~/.claude/skills
ln -s "$(brew --repository dinstein/agenthub)/skills/agenthub" ~/.claude/skills/agenthub
```

A symlink keeps it current: `brew update` pulls this repo and the skill follows.
Copy the directory instead if you would rather pin it.

Then just ask — "add the Linear MCP server and wire it into Claude Code" — and
it will run the same commands documented above, in the right order.

The skill is written against the **released** binary, whose `--help` shows only
the everyday path: `server`, `auth`, `catalog`, `profile`, `client`. The rest
(`secret`, `tool`, `audit`, `daemon`, `session`, `skill`, `doctor`, `config`)
still runs; it is kept off the opening page so a shipped binary recommends one
route rather than listing thirteen. The skill uses those commands where they
are the right answer and says so each time.

## The binary is not notarized

...and that is not what stops an install. The CLI is ad-hoc signed
(`codesign --sign -`), which is not Apple notarization, so `spctl -a` calls it
`rejected`.

That verdict does not mean it is blocked. Gatekeeper acts on files carrying
`com.apple.quarantine`, which quarantine-aware downloaders set — browsers,
AirDrop. Homebrew fetches over curl and does not, so a brew-installed binary
runs directly:

```sh
$ xattr "$(brew --prefix)/bin/agenthub"
com.apple.provenance
$ agenthub --version
agenthub 0.4.0-...
```

No `xattr -cr` is needed here. The GUI is the different case: `AgentHub.app`
is downloaded from a browser, so it does arrive quarantined and needs
right-click → Open the first time.

## Notes for maintainers

`Formula/agenthub.rb` is **generated**, not hand-written: each release renders
it from that release's own `checksums-cli.txt`, so the `sha256` values come
from the artifacts that were actually published rather than from someone
copying six hashes across. Do not edit it by hand — the next release
overwrites the file, and a stale hash fails at install time on somebody else's
machine.

Two paths in the source repository write it, both through
`scripts/homebrew-formula.sh`: the `homebrew` job in
`.github/workflows/release.yml` on a version tag, and
`scripts/release-local.sh` for when Actions cannot run.

Which repository's Releases the download URLs point at is
`HOMEBREW_SOURCE_REPO` in `scripts/homebrew-formula.sh`.

Neither path writes this repository directly. Both hand the finished formula to
`scripts/tap-sync.sh`, which puts it here together with the skill, as **one
commit**. The two are not independent — the formula installs a binary and the
skill documents that same binary's command surface — so a tap where they landed
separately would have a window in which either one described a version the other
did not.

The formula installs a prebuilt binary rather than building from source, for
one reason: `-X main.channel=release` is what makes a shipped binary resolve
the real data directory instead of a development one. A source-building
formula would have to restate that flag, and the day it failed to, every user
of this tap would quietly be running against the development directory and
reporting that their configuration had vanished. The formula's test asks the
binary to prove it:

```ruby
refute_match "(dev)", shell_output("#{bin}/agenthub --version")
```

`skills/agenthub/SKILL.md` is **generated here too**, from
`skills/agenthub/SKILL.md` in the source repository, and it carries a banner
under its frontmatter saying so. Edit it there, not here: a change made in this
repository survives until the next release and then disappears, without ever
showing up in a diff anyone reads.

It stopped being hand-maintained in this repository for the reason a second copy
always does. The document's whole job is to describe the CLI's visible surface,
and this repository has no way to know that surface changed — so it only tracked
it when somebody remembered, in a different checkout, after the release. Twice
nobody did, and the skill went on teaching flags a release build had stopped
accepting.

What has not changed is how it is written: by hand, and re-checked against a
**release** build rather than a `make bin` one, because the two do not advertise
the same commands.
