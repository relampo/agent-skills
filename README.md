# Relampo Agent Skills

Agent skills that teach AI coding agents how to install, author, correlate, validate, and run [Relampo](https://docs.relampo.com) YAML performance tests.

[![skills.sh](https://skills.sh/b/relampo/agent-skills)](https://skills.sh/relampo/agent-skills)

## Install

```bash
npx skills add relampo/agent-skills
```

List skills in this repo without installing:

```bash
npx skills add relampo/agent-skills --list
```

Install only the Relampo skill globally for all detected agents:

```bash
npx skills add relampo/agent-skills -s relampo -g -y
```

This package installs **agent instructions**, not the Relampo CLI binary.

Install the CLI from the official documentation: **https://docs.relampo.com/cli/installation**

Then prepare it:

```bash
relampo setup
relampo version
```

Docs: https://docs.relampo.com

## Skills

### relampo

Teaches agents the Relampo workflow:

`setup → record|converter → correlate → validate → run → distributed run`

Use when you need agents to write Relampo YAML, generate test data with built-in functions, fix dynamic token correlation, script WebSocket / gRPC / WebRTC / SQL steps, drive CSV-parameterized tests, run local or GitHub Actions distributed load, or use Load by Intent.

Targets Relampo YAML contract **1.0.5**.

## Layout

```text
skills/
  relampo/
    SKILL.md
    references/
      yaml-spec.md        # full schema
      builtins.md         # ~180 built-in functions
      data-sources.md     # CSV/TXT parameterization
      protocols.md        # WebSocket, gRPC, WebRTC, SQL
      load-and-intent.md  # load profiles, capacity search
      correlation.md      # tokens, sessions, CSRF
      cli-commands.md     # every command and flag
      editor.md           # editor and Studio behaviour
      distributed.md      # GitHub Actions runs
      gotchas.md          # failure modes
    assets/examples/      # validated starter scripts
```

## Maintaining

See [MAINTAINING.md](MAINTAINING.md) for where the ground truth lives and how to re-sync the skill.

## License

MIT
