# Relampo Agent Skills

Agent skills that teach AI coding agents how to install, author, correlate, validate, and run [Relampo](https://docs.relampo.com) YAML performance tests.

[![skills.sh](https://skills.sh/b/relampo/agent-skills)](https://skills.sh/relampo/agent-skills)

## Install

```bash
bunx skills add relampo/agent-skills
```

List skills in this repo without installing:

```bash
bunx skills add relampo/agent-skills --list
```

Install only the Relampo skill globally for all detected agents:

```bash
bunx skills add relampo/agent-skills -s relampo -g -y
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

`setup → record|converter → correlate → validate → smoke → run → github/gitlab run`

Use it for YAML authoring, built-in data generation, token correlation, HTTP Spark hooks, scoped CSV/TXT data, protocol flows, local load, GitHub/GitLab execution, and Load by Intent.

Baseline: released CLI **v2.10.3**, compatibility contract **1.0.10**. Use exactly one scenario per file. Distributed preparation can commit and push datasets; it requires explicit approval.

## Layout

```text
skills/
  relampo/
    SKILL.md
    references/
      yaml-spec.md        # full schema
      builtins.md         # ~180 built-in functions
      data-sources.md     # CSV/TXT parameterization
      spark.md            # request-local JavaScript hooks
      protocols.md        # WebSocket, gRPC, WebRTC, SQL
      load-and-intent.md  # load profiles, capacity search
      correlation.md      # tokens, sessions, CSRF
      cli-commands.md     # every command and flag
      editor.md           # editor and Studio behaviour
      distributed.md      # GitHub/GitLab publication and reports
      gotchas.md          # failure modes
    assets/examples/      # validated starter scripts
```

## Maintaining

See [MAINTAINING.md](MAINTAINING.md) for where the ground truth lives and how to re-sync the skill.

## License

MIT
