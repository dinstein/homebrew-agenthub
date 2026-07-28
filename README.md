# homebrew-agenthub

Homebrew tap for [agenthub](https://github.com/dinstein/agent-hub) — a local
gateway between AI clients and MCP servers.

```sh
brew tap dinstein/agenthub
brew install agenthub
```

## What is in here

`Formula/agenthub.rb`, and nothing else. It is **generated**, not hand-written:
each release renders it from that release's own `checksums-cli.txt`, so the
`sha256` values come from the artifacts that were actually published rather
than from a human copying six hashes across.

Do not edit it by hand. The next release overwrites the file, and a hand-edited
hash that no longer matches its URL fails at install time on somebody else's
machine — the one place a mistake here is expensive.

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
