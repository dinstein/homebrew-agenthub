# homebrew-agenthub

Homebrew tap for [agenthub](https://github.com/dinstein/agent-hub) — a local
gateway between AI clients and MCP servers.

```sh
brew tap dinstein/agenthub
brew trust dinstein/agenthub     # see below — Homebrew 6 refuses otherwise
brew install agenthub
```

## `brew trust` is not optional

Homebrew 6 will not load a formula from a third-party tap until you have said
you trust that tap. Skipping the middle line above does not produce a warning
you can ignore; the install stops:

```
Error: Refusing to load formula dinstein/agenthub/agenthub from untrusted tap dinstein/agenthub.
Run `brew trust --formula dinstein/agenthub/agenthub` or `brew trust dinstein/agenthub` to trust it.
```

Either spelling works. `brew trust dinstein/agenthub` trusts the whole tap, so
a later formula added here is covered too; `brew trust --formula
dinstein/agenthub/agenthub` trusts exactly this one and nothing else. Prefer
the narrower one if you would rather re-consent when something new appears.

The decision is recorded in `~/.homebrew/trust.json` (or
`$XDG_CONFIG_HOME/homebrew/trust.json`), and `brew untrust` reverses it.

This is Homebrew asking whether you trust **this repository**, not macOS
asking about the binary. The two are unrelated, and the binary side is covered
further down.

## What is in here

`Formula/agenthub.rb` is **generated**, not hand-written: each release renders
it from that release's own `checksums-cli.txt`, so the `sha256` values come
from the artifacts that were actually published rather than from a human
copying six hashes across.

Do not edit it by hand. The next release overwrites the file, and a hand-edited
hash that no longer matches its URL fails at install time on somebody else's
machine — the one place a mistake here is expensive.

`skills/agenthub/SKILL.md` is the opposite: hand-written, and nothing
regenerates it. It teaches an AI client how to drive the CLI — add a server,
store its credentials, run its OAuth login, test it, wire it to a client.

It is written against the **released** binary's command surface, which is
smaller than a development build's. A release `--help` shows only the everyday
path — `server`, `auth`, `catalog`, `profile`, `client` — and keeps the govern
and operate groups off the page. Those commands still run; they are just not
advertised, so a shipped binary recommends one route instead of listing
thirteen.

That distinction is the whole reason this file is hand-maintained. A skill
written against `make bin` would hand users commands their own `--help` never
mentions, and the failure arrives after they have pasted one somewhere that
matters. Where a hidden command is genuinely the right answer (`secret` for an
API key, `audit` to prove a call landed) the skill uses it and says outright
that it is off the help page. When the CLI's visible surface changes, re-check
this file by hand against `bin/agenthub-release`, never `bin/agenthub`.

Install it into Claude Code with:

```sh
mkdir -p ~/.claude/skills
ln -s "$(brew --repository dinstein/agenthub)/skills/agenthub" ~/.claude/skills/agenthub
```

A symlink keeps it current: `brew update` pulls this repo, and the skill
follows. Copy the directory instead if you would rather pin it.

## How it gets updated

Either path in the source repository writes this file:

- `.github/workflows/release.yml`, job `homebrew` — the normal path, on a
  version tag.
- `scripts/release-local.sh` — the same build, run from a laptop, for when
  GitHub Actions cannot.

Both call `scripts/homebrew-formula.sh`, so the formula does not depend on who
cut the release.

## The formula ships a binary, not a source build

On purpose, and for one reason. `-X main.channel=release` is what makes a
shipped binary resolve the real data directory; the default is a development
one. That flag lives in the release build and nowhere else. A source-building
formula would have to restate it, and the day it failed to, every user of this
tap would quietly be running against the development directory and reporting
that their configuration had vanished — a symptom nobody traces back to a
link-time flag.

So the formula installs exactly the artifact CI built, and its test asks the
binary to prove it:

```ruby
refute_match "(dev)", shell_output("#{bin}/agenthub --version")
```

## The binary is not notarized, and that is not what stops the install

The CLI is ad-hoc signed (`codesign --sign -`), which satisfies the loader on
Apple silicon but is **not** Apple notarization. `spctl -a` will call it
`rejected`.

That verdict does not mean the binary is blocked. Gatekeeper acts on files
carrying `com.apple.quarantine`, and that attribute is set by quarantine-aware
downloaders — browsers, AirDrop. Homebrew fetches over curl and does not set
it, so a `brew install` of this formula produces a binary that runs directly:

```sh
$ xattr "$(brew --prefix)/bin/agenthub"
com.apple.provenance
$ agenthub --version
agenthub 0.3.0-...
```

No `xattr -cr` is needed, and a caveat telling you to run one would be telling
you to fix something that is not broken.

The GUI is the different case. `AgentHub.app` is downloaded from a browser, so
it *does* arrive quarantined and does need right-click → Open (or
`xattr -cr /Applications/AgentHub.app`) the first time. The release notes on
each GitHub Release say so.

The one thing that genuinely stops `brew install` is the tap trust prompt at
the top of this file — a Homebrew policy about this repository, unrelated to
macOS code signing.
