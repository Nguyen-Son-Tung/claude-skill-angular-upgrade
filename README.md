# claude-skill-angular-upgrade

A Claude Code skill for upgrading Angular projects from any legacy version to the latest, safely and incrementally.

## What it does

- Analyzes your project (`package.json`, `angular.json`, `tsconfig`) before touching anything
- Upgrades one major version at a time — never skips versions
- Handles Angular Material breaking changes (including `appearance="legacy"` removal, MDC migration, form field sizing)
- Manages RxJS, TypeScript, and SCSS migration
- Always asks for confirmation before changing behavior or replacing dependencies
- Reports clearly after every step

## Install

```bash
npx skills add your-username/claude-skill-angular-upgrade
```

Or manually:

```bash
mkdir -p ~/.claude/skills/angular-upgrade
curl -L https://raw.githubusercontent.com/your-username/claude-skill-angular-upgrade/main/SKILL.md \
  -o ~/.claude/skills/angular-upgrade/SKILL.md
```

## Usage

In Claude Code, just say:

```
upgrade my Angular project
```

or

```
ng update — help me upgrade from v9 to the latest
```

Claude will analyze your project first and guide you step by step.

## Trigger phrases

- "upgrade Angular"
- "ng update"
- "migrate Angular"
- "Angular version upgrade"
- "update Angular Material"
- "Angular breaking changes"
- "nâng cấp Angular"

## License

MIT
