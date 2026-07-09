# Relampo Agent Skills

Agent skills that teach AI coding agents how to install, author, correlate, debug, and run [Relampo](https://docs.relampo.com) YAML performance tests.

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

This package installs **agent instructions**, not the Relampo CLI binary. Install the CLI separately:

```bash
# macOS / Linux
curl -fsSL https://dl.relampo.com/install.sh | bash
relampo setup
relampo version
```

Docs: https://docs.relampo.com

## Skills

### relampo

Teaches agents the Relampo workflow:

`setup → record|converter → correlate → debug → run → distributed run`

Use when you need agents to write Relampo YAML, fix dynamic token correlation, validate scenarios, run local or GitHub Actions distributed load, or use Load by Intent.

## Layout

```text
skills/
  relampo/
    SKILL.md
    references/
    assets/examples/
```

## License

MIT
